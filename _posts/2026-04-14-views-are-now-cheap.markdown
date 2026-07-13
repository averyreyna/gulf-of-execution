---
layout: post
title: Views Are Now Cheap
date: 2026-04-14
permalink: /conjectures/views-are-now-cheap/
categories: conjectures
---

As an undergraduate student taking CS classes with little coding background, I found it difficult to build the intuition needed to discern which technical concepts were connected and how certain programming tasks were related. While this wasn't for lack of trying, I found the ecosystem of learning tools to be disjointed. In a perfect world, I'd be able to collaborate with a couple of my classmates to compile lecture notes, PDFs, and problem sets for a final, spinning up a shareable localhost tool that pulls it all together. This then becomes a shared workspace where annotations, solutions, and even visualizations can uniquely illustrate how concepts interact. Once we're done studying and the final is taken, this study tool dies with no funeral. Although that is too morbid a metaphor, the question still remains: what happens when software can be this specific and this disposable?

Stewing on it more, what I was in search for was not an application in the traditional sense (i.e., making an account, syncing to someone else's cloud, or being built for a user base that isn't me and two classmates right now). The conventional model forces generality: software is expensive to build, so it gets designed for many people across many contexts.

### An Initial Observation

While thinking about local-first software and how it's situated as a (sort of) complement to malleable software, I wrote down:

> "[a] database is structured data, UI is a medium for interacting with that data... most applications are just some variation of that, plus some added business logic" → "If an agent can generate both a structure to some data and a view for said data via a web app, 'software' becomes more a view than a product you [pay for]" → "ephemeral and personal (truly)"

In short, once you strip that away and its associated multitenancy, all you have left is data and a view shaped for some task. This thinking is not new, with relational DBs and SQL rooted in adding a layer of abstraction above physical storage and accessing it via a query language.[^1] However, the interface layer has not seen the same evolution. The way you see and interact with your data is still determined by the encoded assumptions about the median user, which naturally shape how the software makes you think about your own data. For example, a calendar app makes a claim about how you spatially organize time in a year, a word processing document has a set of assumptions about the structure of the writing process, and even a spreadsheet encodes rules about which types of computation should be accessed first. Alan Kay argued decades ago that this was a failure of the medium,[^2] suggesting that computers should be shaped to our thinking rather than appliances we adapt to our thinking; the mental models of other people are frozen into an interface and optimized for this convergence towards the center. As such, the application-as-product model is an artifact of the view layer being too expensive to produce.

### Views Are Now Cheap

The timing of this observation matters in the post-ChatGPT era, since generating views is increasingly becoming a thing of the past. I'm not arguing that average users are creating their own solutions for external tools like Grafana or QuickBooks; rather, the decision to learn development skills becomes more approachable since code generation is cheaper and more abundant than ever. LLMs, on average, collapse the cost for bound, low-stakes, and specific views. This is akin to a whiteboard—the sketching, erasing, and connections between elements don't have to be archived or made more serious than the task at hand. Geoffrey Litt's thoughts on malleable software in the age of LLMs[^3] serve as an anchor here, but I'm focusing specifically on ephemerality rather than persistent customization.

LLMs compress a slow, product-mediated feedback loop, making iteration much quicker, and the app shifts from a fixed product to a flexible view shaped by the user's needs. Once the development loop is tight enough, the "app" in the middle stops being a persistent source of friction that constantly needs updating and becomes something generated, used, and discarded. Similarly, Ink & Switch considers persistent, malleable tools[^4] and extends Kay's framework on software's shape changing over time, but what happens when these tools are allowed to be maximally specific?

### Localhost as the Interface

When generating views is cheap and the data you need lives on your machine, localhost moves towards being a general-purpose interface and away from a strictly developer tool. Developers already use localhost this way: it is where you run, inspect, and iterate before shipping. But the localhost URL has a second property that matters here: it is session-scoped. The server lives as long as the process does, and nothing persists beyond the task that created it. Tools like ngrok extend this further: they expose a local port to a shareable URL for the duration of a session, letting collaborators access the same workspace without provisioning any infrastructure. The study tool from the opening would live at a localhost port, tunneled to classmates, and disappear when we closed our laptops.

<figure>
    <img src="{{ site.baseurl }}/images/alabaster.jpg" loading="lazy" alt="">
    <figcaption>Cropped section of <em>The Artist's Painting-Room</em>, Mary Ann Rebecca Alabaster, 1830. © Art Gallery of Ontario.</figcaption>
</figure>

### Software as a Medium

Given all this, software is becoming an artifact of collaboration shaped by the interaction and disposed of afterward. Similar to the whiteboard analogy, the marks left on display are a residue that indicates the studying process—software can now become what's left over. Compared to custom tooling and vibe coding, this perspective approaches software as something that can be social, situational, and ephemeral first and foremost. For the former, it is built for a role and persists across uses; for the latter, it is entirely focused on producing a product you can deploy regardless of its initial quality.

### Current Tensions

With any new ideas involving agents and LLMs, there are immediate concerns about trust and security. Who should audit the code an LLM wrote that has read access to your filesystem? My initial thoughts are that the trust model has to shift from institutional to personal. When you install an app, trust is outsourced to a review process and the company behind it. When you generate code locally, you become the auditor or you accept the risk of not being one. For a developer, this is manageable: the generated code is readable and the blast radius is your own machine. For a non-technical user, the risk calculus is harder, since they lack the vocabulary to inspect what the tool is actually doing. Sandboxing and containerization can lower this risk, but they add friction back into the loop, partially undoing the cost savings that made ephemeral views attractive in the first place. There is no clean resolution here yet, only a tradeoff that becomes more tolerable as LLM code quality improves and as tooling for sandboxed local execution matures.

There are also problems with schemas, as there is no standard way to point an LLM at your stuff and have it understand its structure. With codebases, there is an innate, searchable architecture afforded to agents by the folder hierarchy across all software repositories. However, with unstructured data, the agent cannot make safe assumptions about context and traversal, making it difficult to properly spin up views to match the cadence at which you want to produce them. As with anything related to software construction, the activation energy between groups varies widely. For CS students and engineers, creating views via LLM-assisted programming is much easier, since they are beginning to acquire the language to describe exactly how data should be manipulated. The more non-technical version of this is further out, but as LLMs progressively get better at writing code more consistently, the explanations for its scaffolding can occur simultaneously.

### Future Research Directions

The view layer is now cheap. What's next? The interesting research directions are two-fold. For starters, we can now consider how to better support ad-hoc view generation, ideally striking a balance among correct persistent context, secure routes to that context, and the standardization of unstructured file data. More broadly, there are questions concerning data access conventions, trust models for generated code, and whether this interface pattern generalizes beyond technical users.

[^1]: Codd's abstractions also act as a prerequisite for my argument. You can't generate views of data if it's not queryable in a structured way. While Codd was making claims in the world of relational databases, his framework can be applied to UI generation, since the idea of "software fundamentally being a view of structured data" has legs to stand on, as articulated by Kay in Smalltalk.

[^2]: Kay really delineated between "doing something with a computer" and "using a computer application." He had a vision for Smalltalk rooted in the idea that computers should be a medium for human thought, not just machines with executable instructions. The environment wasn't just a programming language; it was a live system that let the user point at anything on the screen to change it and inspect its behavior.

[^3]: Litt argues that LLMs could enable a new era of end-user programming, where people create and modify their own tools without traditional programming skills. His frame is largely about persistent customization, software that bends to the user over time and accumulates their preferences. This post takes a different fork: rather than tools that grow alongside a user, the interesting case is tools that exist only for a task and are discarded once it ends. The distinction matters because persistent customization still implies a product with a lifecycle; ephemeral views remove the lifecycle constraint entirely.

[^4]: Ink & Switch extends Kay's original vision by asking what it would take for users to reshape their tools in the moment, not by filing a feature request, but by directly modifying behavior. Their work explores systems where composable, inspectable pieces replace monolithic applications, and tools evolve alongside a user's thinking. The difference from this post's framing is scope: Ink & Switch is interested in tools that are malleable and durable. The ephemerality argument adds a third option, tools that are maximally specific precisely because they are not trying to last.
