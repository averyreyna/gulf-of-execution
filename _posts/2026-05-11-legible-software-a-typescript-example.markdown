---
layout: post
title: "Legible Software: A TypeScript Example"
date: 2026-05-11
permalink: /conjectures/legible-software-a-typescript-example/
categories: conjectures
---

Daniel Jackson's work on concept-based software design proposes a simple structural rule: each piece of behavior should live in its own concept, or an independent state machine with its own state and actions. Concepts never reference each other's internals and the *only* way to wire them together is through declarative rules that say "when this action in concept called synch A fires, trigger this action in concept B" called synchronizations. The payoff is legibility: you can understand a concept by reading its actions, and you can understand the whole system by reading its syncs, without opening anything else.

Here is what that looks like in practice using authentication in a TypeScript codebase as an example. The conventional approach with Express and Prisma puts everything together:

```typescript
// routes/auth.ts
router.post('/register', async (req, res) => {
  const { email, password } = req.body;

  const salt = await bcrypt.genSalt(10);
  const hashedPassword = await bcrypt.hash(password, salt);

  const user = await prisma.user.create({
    data: { email, password: hashedPassword, salt }
  });

  res.json({ user });
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({ where: { email } });
  const valid = await bcrypt.compare(password, user.password);

  if (!valid) return res.status(401).json({ error: 'Invalid password' });
  res.json({ user });
});
```

Password hashing appears in two places with nothing preventing a third. The `password` and `salt` columns live on the `User` table, so authentication is entangled at the schema level before any code is written. Each route handler knows about `bcrypt` directly. To change how passwords are stored, you have to find and touch every place that knows about it.

The concept approach pulls password logic into its own module with a single point of entry:

```typescript
// concepts/Password.ts
const PasswordConcept: Concept = {
  state: {
    passwords: new Map<string, { hash: string; salt: string }>()
  },

  async execute(action, input) {
    switch (action) {

      case 'set': {
        const { user, password } = input;
        const salt = await bcrypt.genSalt(10);
        const hash = await bcrypt.hash(password, salt);
        this.state.passwords.set(user, { hash, salt });
        return { user };
      }

      case 'check': {
        const { user, password } = input;
        const stored = this.state.passwords.get(user);
        if (!stored) return { error: 'User not found' };
        const valid = await bcrypt.compare(password, stored.hash);
        return { valid };
      }

    }
  }
};
```

`Password.set` is the only door into that state. Aliasing across concept boundaries is structurally impossible, not just a convention. The decision to hash and store a password when a user registers, which in the traditional codebase lives inside a route handler mixed with HTTP parsing and database calls, is now a separate, named rule.

```typescript
const registrationSync: SyncRule = {
  name: 'NewPassword',
  when: [{ concept: 'User', action: 'register', outputPattern: { user: '?user' } }],
  then: [{ concept: 'Password', action: 'set', input: { user: '?user', password: '?password' } }]
};
```

The person writing this sync never opens `User` or `Password`. The output of `User/register` includes a `user` field. The input of `Password/set` accepts one. The signatures match, so you compose. Adding a social login path means adding a new sync, not modifying the existing registration flow. Nothing breaks because nothing was touched.
