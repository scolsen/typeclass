# Higher Kinds

**Note: This is not an officially supported Google product.**

Support for higher kinds in Carp, using the method described in Yallop and
White's
[Lightweight Higher-Kinded Polymorphism](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf)

## Pseudo-Typeclasses

If you're familiar with the [Haskell]() programming language, you've likely
encountered its *typeclasses*--a mechanism that makes it possible to program
with a high degree of abstraction. Functions can be defined entirely in terms of
a small set of primitive functions defined by typeclasses, and any type that implement the typeclasse's primitives works with the functions that use them for free.

This library makes it possible to define constructs roughly equivalent to
typeclasses, and enables the same level of abstraction while remaining true to
the types of typeclass functions, which usually take polymorphic constructors as
arguments. Here's a quick example of the library in action:

```clojure
(load "functor.carp")

(defn add-one [x] (+ x 1))

(deftype (Box a) (Of [a]) (Empty []))

(Kinds.kind Box)
(Functor.derive-fmap Box)
(Functor.instance Box)

(Box.construct (fmap &add-one (Box.functor (Box.of 2))))
;; => (Box.Of 3)
;; Functor.replace is defined as:
;; [x c] (fmap &(const x) c)
(Box.construct (Functor.replace \a (Box.Functor (Box.Of 4))))
;; => (Box.Of \a)

;; When we declared the Box instance, the second argument, `true` indicated
;; we wanted an 'obvious' implementation of fmap derived for us. 
;; We can also handwrite custome implementations wherever need, but not against
;; types you might expect.

(Kinds.kind Maybe)
(Functor.derive-fmap Maybe)
(Functor.instance Maybe)
(defmodule Maybe
  (defn fmap [f maybe-constructor] 
    (match maybe-constructor
      (Constructor.App x b) 
        (match b
          (MaybeBrand.Just) (Constructor.App (~f x) (MaybeBrand.Just))
          _ (Constructor.App x (MaybeBrand.Nothing)))))
)

;; This variant of fmap behaves correctly, plus, it's saved us an extra function
;; call
(Maybe.construct (fmap &add-one (Maybe.functor (Maybe.Just 2))))
;; => (Maybe.Just 3)
```
## Detailed Explanation

TODO: Clean up this section, and the doc as a whole.

While using this library allows you to write typeclass-esque code in Carp and
have it pretty much work as you'd expect, there are a few things to keep in
mind:

1. Injections and projections (`construct` and `constructor`) always need to be
   called before a chain of typeclass function calls.

   This is just an unfortunate reality because of the way this system works.
   Since Carp doesn't support polymorphic type constructors within its type
   system yet, we need to define typeclasses in terms of concrete types that
   encode a suspended type application. Thus, in order to use these functions,
   and in order to extract their results, we need to call explicit projections
   and injections from base types into the abstract `Constructor` type.
   Basically, all typeclass reliant code needs to have the form:

   ```clojure
   (MyType.construct (...a bunch of fmap calls etc operating on (App's) (MyType.constructor (MyType.SomeValue 2))))
   ```

   Most typeclasses will alias `constructor` for ease of use, e.g. Functor
   instances define `MyType.functor` which is just an alias for construct.

2. `construct` and `constructor` preserve typing. Since brands are unique, you
   can't use the `Constructor` type for any type-invalidation funny business,
   for example:

    ```clojure
    (Maybe.construct (Functor.replace \a (Box.functor (Box.Of 4))))
    ```

    Results in an error complaining about the brand mismatch:

    ```
    I can’t match the types `BoxBrand` and `MaybeBrand` within `(Functor.replac ...  4)))`.

    (Box.functor (Box.Of 4)) : (Constructor r7 BoxBrand)
    At line 1, column 38 in 'REPL'

    Expected second argument to 'Functor.replace' : (Constructor r4 MaybeBrand)
    At line 1, column 19 in 'REPL' 
    ```

   Of course, should you want to define transforms between Box and Maybe that
   make use of typeclasses, you certainly can do so. You'd need to define a
   function that transforms a Constructor's Brand:

   ```
   (Fn [(Constructor a b)] (Constructor a c))
   ```

   Then, the `construct` function for the appropriate type will be valid.

   Similar comments hold for `constructor`--it will result in type errors when
   you call the incorrect variant on a base value, thanks to brand uniqueness:

   ```
   (fmap &p (Maybe.constructor (Box.Of 4)))
   ```
  
   Will spit out

   ```
   I can’t match the types `(Box r8)` and `(Maybe Int)` within `(Maybe.functor  ... f 4))`.

   (Box.Of 4) : (Box r8)
   At line 1, column 25 in 'REPL'

   Expected first argument to 'Maybe.functor' : (Maybe Int)
   At line 1, column 11 in 'REPL'
   ```
  
  reasonably so.
