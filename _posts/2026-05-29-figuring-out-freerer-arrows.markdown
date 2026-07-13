---
layout: post
title: Figuring Out Freerer Arrows
date: 2026-05-29
permalink: /conjectures/figuring-out-freerer-arrows/
categories: conjectures
---

Monads are essentially a design pattern for sequencing computations that have side effects. It acts as a "wrapper" around a value that carries some "context," plus some rules for chaining operations that respect it. As a developer, it might be your intuition to think that monads are an alternative to running multiple computations across scatered functions, sort of connecting it to error and type safety. However, they're more about effect management and composition, ensuring that these said effects are explicit in a type system and composable in a principled way.

In terms of coding parallels, `Maybe`/`Optional`, `Promise<T>`, and `flatMap` on arrays are similar to monads conceptually. Here are some comparisons:

**Maybe/Optional — "this might not exist"**

Imperative (JS without monads):

```javascript
function getCity(user) {
  if (user !== null) {
    if (user.address !== null) {
      if (user.address.city !== null) {
        return user.address.city;
      }
    }
  }
  return null;
}
```

Monadic (using optional chaining, which JS borrowed from FP):

```javascript
const city = user?.address?.city ?? "unknown";
```

The `?.` operator *is* basically the Maybe monad. The "null" context propagates through the chain automatically — if any step is null, the whole thing short-circuits.

**Promise — "this value arrives later"**

```javascript
fetchUser(id)
  .then(user => fetchOrders(user.id))    // only runs if fetchUser succeeded
  .then(orders => formatOrders(orders))  // only runs if fetchOrders succeeded
  .catch(err => showError(err));         // catches failure anywhere in the chain
```

The async/rejected context flows through every `.then()`. You never manually check "did the previous step finish?" since the monad handles that for you. Compare to callback hell we would have without it:

```javascript
fetchUser(id, function(err, user) {
  if (err) return showError(err);
  fetchOrders(user.id, function(err, orders) {
    if (err) return showError(err);
    formatOrders(orders, function(err, result) {
      if (err) return showError(err);
      // finally do something
    });
  });
});
```

Same logic, but now error handling is manually repeated at every step. The monad eliminates that repetition.

**flatMap — "this one thing produces many things"**

```javascript
const items = users
  .flatMap(user => user.orders)
  .flatMap(order => order.items);
```

Each `flatMap` is bind. Without it, you need nested loops that grow with each level:

```javascript
const items = [];
for (const user of users) {
  for (const order of user.orders) {
    for (const item of order.items) {
      items.push(item);
    }
  }
}
```

The structure is the same as the Promise case—each step produces a container (an array instead of a Promise), and flatMap sequences them without manual unwrapping.

While this structure is expressive, there is a limitation to how opaque we can make its operations—as pointed out by VanDomelen et al.[^1] in their *Freer Arrows* paper. To make this premise moer concret, the authors consider building an EDSL for web services in Haskell. They define the effects as a GADT and model them with a freer monad:

```haskell
data FreerMonad (e :: Type -> Type) (a :: Type) where
  Ret :: a -> FreerMonad e a
  Eff :: e x -> (x -> FreerMonad e a) ->
          FreerMonad e a

data WebServiceOps :: Type -> Type where
  Get  :: URL -> [String] -> WebServiceOps String
  Post :: URL -> [String] -> String -> WebServiceOps ()

get :: URL -> [String] ->
        FreerMonad WebServiceOps String
get url params = Eff (Get url params) Ret

post :: URL -> [String] -> String ->
         FreerMonad WebServiceOps ()
post url params body = Eff (Post url params body) Ret
```

We can interpret these into any monad by supplying an effect handler:

```haskell
interp :: Monad m => (forall a. e a -> m a) ->
                      FreerMonad e a -> m a
interp _      (Ret a)       = return a
interp handle (Eff eff k) =
  handle eff >>= interp handle . k
```

The problem surfaces when we try to write a static analyzer as a function inside Haskell itself. Suppose we want to count the number of post operations, or collect all URLs being posted to. Consider this function:

```haskell
postDepends :: URL -> [String] -> String ->
                FreerMonad WebServiceOps ()
postDepends url params body =
  get url params >>=
  postNTimes url params body .
  (read :: String -> Integer)
```

Here, the number of post operations depends on the result of the get operation at runtime. We can only know it *after* actually performing the get. Now consider:

```haskell
echo :: URL -> URL -> [String] ->
         FreerMonad WebServiceOps ()
echo url1 url2 params =
  get url1 params >>= post url2 params
```

In this case, the URLs are statically visible in the code—there is one get and one post, and neither depends on an effectful result. But because monads are so expressive, there is no way to write a static analyzer that can distinguish this case from `postDepends`. The dynamic nature of bind means we cannot inspect the structure of the computation without running it.

### Freer Arrows for Static Analysis

Freer arrows are a relatively expressive structure that is amenable to static analysis, offering an alternative trade-off between expressiveness and static analyzeability compared to freer monads.

To revisit the web service example, the `WebServiceOps` datatype is redefined with an additional type argument for the input:

```haskell
data WebServiceOps :: Type -> Type -> Type where
  Get  :: URL -> [String] -> WebServiceOps () String
  Post :: URL -> [String] -> WebServiceOps String ()
```

Compared with the `WebServiceOps` shown above, this datatype additionally contains the type of the input as its first type argument. We can further define the following "smart constructors" for `Get` and `Post`:

```haskell
embed :: e x y -> FreerArrow e x y
embed f = Comp (,()) f (arr fst)

get :: URL -> [String] ->
        FreerArrow WebServiceOps () String
get url params = embed $ Get url params

post :: URL -> [String] ->
         FreerArrow WebServiceOps String ()
post url params = embed $ Post url params
```

The `embed` function "embeds" an effect in a `Comp`. `Comp` requires an existential type—*i.e.*, the type of the carried value—but we do not need to carry any values in this case, so we just use `()` as that type. Then `get` and `post` are defined as embedding `Get` and `Post`, respectively. We can now re-implement the `echo` function shown previously using freer arrows:

```haskell
echo :: URL -> URL -> [String] ->
         FreerArrow WebServiceOps () ()
echo url1 url2 params =
  get url1 params >>> post url2 params
```

[^1]: VanDomelen et al.'s paper covers freer arrows in much more depth than this post does, including three distinct variants: freer pre-arrows, freer arrows, and freer choice arrows—along with a full case study in choreographic programming using the HasChor framework. This post focuses on the core motivation for why the structure is worth reaching for in the first place.
