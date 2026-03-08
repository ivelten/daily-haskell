+++
title = "Deconstructing Optics: A Profunctor Approach with Prolens"
date = 2026-03-08T02:28:04Z
draft = false
slug = "deconstructing-optics-a-profunctor-approach-with-prolens"
tags = ["haskell", "functional-programming", "optics"]
+++


Managing deep, immutable state in C# often leads to "nesting hell," where updating a property inside a deeply nested object requires a cascade of manual object cloning or the use of heavy-handed libraries like `AutoMapper` or `DeepCopy`. In Haskell, we solve this with optics—specifically lenses—but the sheer breadth of the `lens` library can be overwhelming for teams migrating from object-oriented backgrounds. The `prolens` package provides a minimalist, high-performance, and educational alternative that strips optics down to their category-theoretic essence: the profunctor.

## The Profunctor Core

At its heart, a lens is a way to focus on a piece of data within a larger structure. If you are coming from C#, think of a lens as a bidirectional property accessor. In `prolens`, we define a lens using the `Profunctor` and `Strong` type classes.

```haskell
-- A simplified representation of the profunctor lens type
type Lens s t a b = forall p. Strong p => p a b -> p s t
```

Here, `s` is the structure, `t` is the modified structure, `a` is the focused part, and `b` is the modified part. By using `Strong`, we gain the ability to distribute the lens over a tuple, effectively allowing us to "keep" the rest of the structure while we transform the focus.

## Practical Composition

The power of lenses lies in composition. In a typical C# application, accessing a nested property looks like `user.Address.Street`. In Haskell, if we define lenses for these fields, we can compose them using the `.` operator.

```haskell
import Prolens

data User = User { _name :: String, _address :: Address }
data Address = Address { _street :: String }

-- Using lens-like creation patterns
addressL :: Lens' User Address
addressL = lens _address (\u a -> u { _address = a })

streetL :: Lens' Address String
streetL = lens _street (\a s -> a { _street = s })

-- Composition: Focus on the street name directly
userStreetL :: Lens' User String
userStreetL = addressL . streetL
```

Unlike C# property accessors, which are tightly coupled to the object instance, these lenses are first-class functions. You can pass them as arguments, store them in lists, or compose them dynamically at runtime.

## Why Profunctors?

The choice of profunctors over the traditional Van Laarhoven (CPS-based) encoding used in the heavy `lens` library is a strategic trade-off. Van Laarhoven lenses rely on `Const` and `Identity` functors to trick the type system into performing `get` and `set` operations. While elegant, they can produce massive type-level signatures that are difficult for beginners to debug.

Profunctor-based optics, as implemented in `prolens`, are more "honest." They map directly to the transformation of a function. A `Profunctor p` is a mapping that supports `dimap :: (a' -> a) -> (b -> b') -> p a b -> p a' b'`. This makes the logic of "zoom-in" and "zoom-out" explicit in the types.

## Common Pitfalls and Performance

A common pitfall for those transitioning from C# is treating lenses as "pointers." In C#, if you get a reference to a property, you might mutate it in place. In Haskell, lenses do not mutate; they return a new copy of the structure. When dealing with large records, this might seem inefficient. However, because Haskell uses structural sharing, updating a single field in a large record does not copy the entire object tree—it only copies the spine of the record leading to the modified field.

One edge case to watch for is "lens fragmentation." If you create too many granular lenses, you may find yourself struggling with type inference. Always prefer `Lens'` (a lens where the structure doesn't change type) over the more general `Lens` unless you are performing type-level transformations.

## Trade-offs

`prolens` is not a drop-in replacement for the full `lens` library. It lacks the massive suite of operators (`^.`, `.~`, `%~`, `%%~`) and the advanced support for traversals and isomorphisms. However, for a senior engineer, this is a feature, not a bug. By using `prolens`, you force your team to understand *how* the optics work rather than relying on a "magic" library that handles complex state updates implicitly.

If your project requires high-performance optics with minimal compile-time overhead, `prolens` is an excellent choice. It provides the necessary primitives to build what you need without the cognitive load of a 50,000-line dependency.

## Further Reading

- [Kowainik/prolens GitHub Repository](https://github.com/kowainik/prolens)
- [Profunctor Optics: The Essence of Lenses](https://www.cs.ru.nl/~jmh/papers/profunctor-optics.pdf)
- [The lens library vs. profunctor optics](https://hackage.haskell.org/package/lens)
