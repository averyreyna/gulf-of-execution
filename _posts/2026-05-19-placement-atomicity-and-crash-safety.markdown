---
layout: post
title: Placement, Atomicity, and Crash Safety
date: 2026-05-19
permalink: /conjectures/placement-atomicity-and-crash-safety/
categories: conjectures
---

Most of the interesting decisions in a resource scheduler aren't about the happy path. They're about what happens when two placement requests land simultaneously, when the process crashes halfway through committing a workload, and when a host's capacity slowly drifts from what the scheduler thinks it is. The naive implementation of each of these looks fine in testing and fails in production in ways that take a while to diagnose.

### Placement

Round-robin and least-loaded are the intuitive starting points. Round-robin distributes requests by turn, ignoring every resource dimension. It works until hosts diverge in capacity, at which point it routes large requests to hosts that can't fit them. Least-loaded is more considered—pick the host with the most available capacity—but it has a fragmentation problem that only shows up at scale.

Imagine four hosts each provisioned with 8 CPU cores. After some placements, three hosts have 3 cores free and one has 7. Least-loaded correctly sends the next large request (say, 6 cores) to the host with 7. Then a stream of 1-core requests comes in. Least-loaded spreads them evenly across all four hosts. The result is four hosts sitting at 2–3 cores free each: enough for another small request, not enough for a large one. The capacity exists in aggregate but it's been scattered across the fleet, and any request that needs more than 3 cores now has nowhere to go.

Best-fit inverts this. Pack requests as tightly as possible onto as few hosts as possible. A nearly-full host stays full; an empty one stays empty and available for requests that need the room. The scoring function computes total slack and the tightest fit wins. This is essentially the bin packing problem, and best-fit is one of the classical approximation heuristics for it.[^1]

The complication is that CPU, memory, and disk aren't comparable units. A 0.5-core CPU difference and a 500MB memory difference don't mean the same thing, and summing them naively produces a number dominated by whichever resource is measured in the largest unit. Normalizing to a common unit gets you into the right ballpark:

```go
type Host struct {
    ID     string
    CPU    float64
    Memory int64
    Disk   int64
}

type Request struct {
    CPU    float64
    Memory int64
    Disk   int64
}

const gb = 1 << 30

func slack(h Host, req Request) float64 {
    return (h.CPU - req.CPU) +
        float64(h.Memory-req.Memory)/gb +
        float64(h.Disk-req.Disk)/gb
}

func place(hosts []Host, req Request) (Host, bool) {
    var best *Host
    for i := range hosts {
        h := &hosts[i]
        if h.CPU < req.CPU || h.Memory < req.Memory || h.Disk < req.Disk {
            continue
        }
        if best == nil || slack(*h, req) < slack(*best, req) {
            best = h
        } else if slack(*h, req) == slack(*best, req) && h.ID < best.ID {
            best = h
        }
    }
    if best == nil {
        return Host{}, false
    }
    return *best, true
}
```

But normalization only partially solves the problem. Each resource has a different overcommit profile: CPU is time-shared, so 2–4x overcommit is standard. Memory overcommit risks OOM kills and is kept conservative, usually around 1.2x. Disk thin-provisioning can go further than either. A flat normalized score treats these as equivalent constraints when they aren't, which means the scheduler can mislabel a "best fit" in environments with non-uniform resource pressure. The more accurate version weights each dimension by its overcommit tolerance, which changes which host wins depending on what the request is asking for. Tie-breaking by host ID keeps selection deterministic across runs with identical state, which matters less for correctness than for being able to replay placement decisions when debugging.

### Atomicity

Selecting a host is a read, reserving capacity is a write, and the gap between those two operations is where double-booking happens. The race is: two goroutines both read a host showing 4 CPU cores free. Both check whether their 3-core request fits. Both pass. Both write back 1 core remaining. The host is now over-committed by 3 cores and neither goroutine knows. It's the check-then-act problem under concurrency, and it's easy to miss when the reservation logic is first written against a single-threaded test harness where nothing races.

A mutex around the check-and-write solves it. It also requires every code path that touches host state to participate in the same locking protocol, and that discipline tends to erode as a codebase grows and new paths are added by people who weren't there for the original design. BoltDB's `db.Update` is cleaner: it wraps a write transaction that runs exclusively, one at a time, globally. The read, check, and write are serially isolated without any application-level coordination required:

```go
err := db.Update(func(tx *bolt.Tx) error {
    b := tx.Bucket([]byte("hosts"))

    raw := b.Get([]byte(hostID))
    var h Host
    if err := json.Unmarshal(raw, &h); err != nil {
        return err
    }
    if h.CPU < req.CPU || h.Memory < req.Memory {
        return ErrNoCapacity
    }

    h.CPU -= req.CPU
    h.Memory -= req.Memory

    updated, _ := json.Marshal(h)
    return b.Put([]byte(hostID), updated)
})
```

`ErrNoCapacity` rolls the transaction back with no capacity consumed. A mid-update crash doesn't partially commit—BoltDB's write-ahead log makes the transaction atomic with respect to failures. The host record is either before the reservation or after it.[^2]

The tradeoff is throughput. A single write lock serializes all host updates globally, which is a bottleneck if placement frequency is high enough to saturate it. For most orchestrators, placement isn't the hot path and BoltDB's model is the right call. A system under heavier write pressure would want partitioned locks by host, optimistic locking with retry, or a database with row-level locking semantics.

### Crash Safety

Placing a workload is a multi-step process: reserve resources on the host, allocate the workload, start the process, update the machine record to running. A crash at any point before the terminal state leaves the host holding reserved capacity for a workload that never finished starting. Those resources need to come back.

The instinct is to add cleanup code to each transition that can fail. Reserve fails—release nothing, nothing was taken. Allocate fails—release the reservation. Start fails—release the reservation and tear down the allocation. I've worked in codebases where this pattern was applied consistently early on and then slowly drifted as new transitions were added, leaving host records that diverged from reality over time: reserved capacity that was never returned, machines that couldn't be placed because the scheduler thought the host was full when it wasn't.

A finalizer fires unconditionally when the machine exits the state machine, regardless of how:

```go
machine.OnFinalize(func(ctx context.Context, m *Machine) error {
    if m.HostID == "" {
        return nil
    }
    return releaseResources(ctx, m.HostID, m.Requested)
})
```

If resources were allocated, they're released. If they weren't—the crash happened before reservation—`m.HostID` is empty and the function returns early. Adding a new transition means writing its logic, not revisiting the cleanup path.

There's still a gap. If the process crashes after `releaseResources` returns but before the state machine records the terminal state, crash-resume will invoke the finalizer again against a host that already had its capacity restored. The resources get double-released. Fixing it requires making the resource update and the state transition a single atomic database write—either both commit or neither does. Without that, the narrow window between those two operations remains a potential inconsistency, and how much that matters depends on how often the process actually crashes and how quickly it's detected.

[^1]: Best-fit is one of several classical heuristics for the bin packing problem, which asks how to pack items of varying sizes into the fewest bins of fixed capacity. The problem is NP-hard, so practical schedulers use approximation heuristics like best-fit, first-fit, and first-fit decreasing. The connection is worth making explicit because bin packing has a substantial theoretical literature, including approximation ratio bounds, that can inform how you reason about scheduler behavior in adversarial cases.

[^2]: BoltDB is a single-writer database by design, which distinguishes it from SQLite in WAL mode, where concurrent reads are allowed alongside a single writer. Both would solve the double-booking problem here, but BoltDB's model is simpler to reason about: there is no concurrent write access, period. The cost is that BoltDB doesn't support SQL, so any querying beyond key lookups requires scanning buckets manually.
