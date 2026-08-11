---
title: OCaml Functors
type: language
created: 2026-08-11
tags: [ocaml, modules, functors, parametric-modules, abstraction]
---

# OCaml Functors

## Junior — What even is this?

Imagine you write a `Set` library. Sets need to compare elements — but you
don't know in advance whether the elements will be integers, strings, or your
custom `User` type.

A **functor** is a *module that takes another module as input and produces a
new module as output*. Think of it as a function, but at the module level.

```ocaml
(* A module signature: "I need something comparable" *)
module type Comparable = sig
  type t
  val compare : t -> t -> int
end

(* A functor: give me a Comparable, I give you a Set *)
module MakeSet (Elem : Comparable) = struct
  type t = Elem.t list   (* naive impl *)

  let empty = []

  let add x s =
    if List.exists (fun e -> Elem.compare e x = 0) s
    then s
    else x :: s

  let mem x s =
    List.exists (fun e -> Elem.compare e x = 0) s
end
```

Using it — you "apply" the functor to a concrete module:

```ocaml
module IntSet = MakeSet(struct
  type t = int
  let compare = Int.compare
end)

let s = IntSet.(empty |> add 3 |> add 1 |> add 3)
(* s = [1; 3] — 3 not duplicated *)
```

Key insight: `MakeSet` is not a module — it is a *function from modules to
modules*. `IntSet` and `StringSet` are distinct types; the compiler keeps
them separate.

---

## Mid-level — Real patterns

### The `Map.Make` / `Set.Make` pattern

OCaml's stdlib uses functors everywhere. `Map.Make` is the canonical example:

```ocaml
module StringMap = Map.Make(String)

let m = StringMap.(empty |> add "a" 1 |> add "b" 2)
let v = StringMap.find "a" m  (* : int *)
```

`String` satisfies `Map.OrderedType` because it has `type t = string` and
`val compare : t -> t -> int`.

### Functor signature — sealing the output

You can annotate the output type to hide implementation details:

```ocaml
module type SET = sig
  type elt
  type t
  val empty : t
  val add   : elt -> t -> t
  val mem   : elt -> t -> bool
end

module MakeSet (Elem : Comparable) : SET with type elt = Elem.t = struct
  (* ... same body ... *)
end
```

`with type elt = Elem.t` is a *type sharing constraint* — it re-exports the
element type so callers can still name it.

### Chaining functors

Functors compose. A balanced-BST functor might wrap `MakeSet`:

```ocaml
module MakeBalanced (Elem : Comparable) =
  MakeBalanced_impl(MakeSet(Elem))
```

### First-class modules (OCaml ≥ 3.12)

Functors are second-class by default (compile-time only). First-class modules
let you pack/unpack at runtime:

```ocaml
let make_set (type a) (module Elem : Comparable with type t = a) =
  (module MakeSet(Elem) : SET with type elt = a)
```

This enables heterogeneous containers and plugin systems.

---

## Senior — Trade-offs and implications

### Strengths

- **Type safety across instantiations** — `IntSet.t` and `StringSet.t` are
  incompatible at compile time. No runtime tag needed.
- **Zero-cost abstraction** — functor application happens at compile time;
  generated code is monomorphised and fully inlined by the compiler.
- **Structural, not nominal** — any module satisfying the signature works,
  without `implements` declarations.

### Weaknesses

| Problem | Detail |
|---|---|
| Verbosity | Each use site needs an explicit `module X = Functor(Arg)` |
| No implicit resolution | Unlike Haskell typeclasses, OCaml won't auto-pick a `Comparable` |
| Sealing boilerplate | `with type` constraints multiply with nesting depth |
| First-class cost | Packing/unpacking first-class modules adds syntax overhead |

### Functors vs typeclasses (Haskell/Rust)

OCaml functors are explicit and structural. Rust traits and Haskell typeclasses
are implicit and nominal — the compiler searches for an instance. OCaml's
approach makes the dependency graph obvious at a cost of verbosity. Neither is
strictly better; the tradeoff is explicitness vs. ergonomics.

### Functors vs interfaces (Java/TypeScript)

Interfaces act on *values*. Functors act on *modules*, which can carry
**types** (not just values). `Map.Make(Key)` produces a map whose key type is
*statically* `Key.t` — no casting, no `any`. This is richer than a generic
class `Map<K>` because the module can export associated types, sub-modules,
and exceptions, not just methods.

---

## Principal — When to use vs. not

### Use functors when

- Building data structures parametrised over element behaviour
  (ordering, hashing, equality).
- Abstracting over storage backends, serialisers, or loggers — pass a module,
  get back a domain module.
- Writing a library that must work with user-defined types without a
  runtime dispatch overhead.

### Avoid functors when

- The parametrisation is over a *value*, not a type or behaviour. Just use a
  higher-order function.
- The abstraction boundary is temporary (inside a single file). A local
  `module` definition or a closure is simpler.
- Team members are unfamiliar with OCaml's module system — the `with type`
  constraint syntax has a steep learning curve.

### Boundary decisions

Expose functor signatures (`module type`) in `.mli` files, never the functor
body. Consumers depend on the signature; your internal representation can
change freely.

If a library surface area is large, consider a *layered functor* approach:
one functor for the core (ordering), a separate one for extensions (pretty
printing, serialisation). This keeps instantiation cheap and lets users opt
in.

---

## See also

- [[ocaml-modules]] — module basics before tackling functors
- [[30-patterns/ports-and-adapters]] — functors are OCaml's native
  ports-and-adapters mechanism
- [[10-concepts/parametric-polymorphism]]
