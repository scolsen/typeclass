# Typeclasses

**Note: This is not an officially supported Google product.**

Typeclasses for the Carp programming language, inspired by Haskell's categorical
classes, implemented as interfaces.

## Copying and Mutating Interfaces

While this library takes significant inspiration from Haskell and attempts to
follow its use of categorical concepts directly in many ways, it harbors no
illusions about functional purity. Carp, unlike Haskell, is not a purely
functional programming language and uses a borrow checker to permit "impurities"
such as side-effects and mutations safely.

In adherence to this reality, you'll find that, in addition to the "purely"
functional variants of interfaces for functor, applicative, and other classes,
there are *mutating* versions which irrevocably alter their arguments
*in-place*. For example, in addition to `fmap`, this library defines `fmap!`:

```clojure
(definterface fmap! (Fn [(Ref (Fn [a] a)) (Ref (f a))] Unit))
```

Which is the *mutating* variant of `fmap` and will apply its function argument
to a structure in-place. 

The library's function names adhere to the convention of appending a `!` to any
interface or function that performs a mutation or alters a value in-place.
Additionally, most if not all of such functions return `Unit`.

Implementations of mutating interfaces such as `fmap!` are generally expected to
perform mutation to alter a value *in-place* and may *potentially be unsafe*.
Contrarily, functions without the trailing `!`, such as `fmap` are "pure", are
generally expected to have no side-effects, and are expected to copy a value or
return an entirely new value rather than modify an argument in-place. 

### Values and References

In keeping with the semantics described above, copying interfaces generally
accept a *value* as an argument, while mutating interfaces generally accept a
*reference*:

```clojure
;; *copying* fmap, takes a value of `(f a)`
(definterface fmap (Fn [(Ref (Fn [a] b)) (f a)] (f b)))
;; *mutating fmap* takes a reference to an `(f a)`
(definterface fmap! (Fn [(Ref (Fn [a] a)) (Ref (f a))] Unit))
```

### Epimorphic and Endomorphic Function Arguments

The copying and mutating variants of interfaces also expect different kinds
of function arguments. Copying interfaces typically only require that their
function arguments be surjective or epimorphic; that is, they are allowed to
have *distinct* argument and return types. Consider the copying variant of
`fmap`:

```clojure
(definterface fmap (Fn [(Ref (Fn [a] b)) (f a)] (f b)))
```

Its function argument maps from the argument type `a` to the return type `b`,
and `a` and `b` may be different; e.g. `Int` and `Bool`.

Contrarily, the function arguments of mutating interfaces *must* be
*endomorphic*; that is, their argument and return types *must* be the same,
consider the mutating variant of `fmap`, `fmap!`:

```clojure
(definterface fmap! (Fn [(Ref (Fn [a] a)) (Ref (f a))] Unit))
```

Its function argument maps from the argument type `a` to the same return type
`a`; e.g. if `a` is an `Int` the function *must* return and `Int`. One could
call this the *endomorphism restriction* for mutating interfaces.

The endomorphism restriction may make more sense when we consider an example use
case of the mutating version of `fmap`:

```clojure
(let-do [m (Maybe.Just 2)] (fmap! &inc &m) m)
```

In the above sample, we use `fmap!` to increment the value contained in a Maybe
in-place. `&m` is a reference to some memory *allocated to hold a value of
`Maybe Int`*. The call to `fmap!` directly modifies this reference to update the
value held in the memory it points to. If we altered the inhabitant type of the
maybe, this might cause memory errors, as the size allocated for the value may
wind up being incorrect once the type changes.

The following sections briefly described the "classes" defined in the library.

## Functor

Generic operations over functors are defined in the `Functor` module. To utilize
these functions, a type must implement:

- `fmap` to use any *copying* style functions (not ending in `!`)
- `fmap!` to use any *mutating* style functions (ending in `!`)

## Applicative

Generic operations over applicatives are defined in the `Applicative` module. To
utilize these functions, a type must implement:

- `pure`
- `sequence` to use any *copying* style functions (not ending in `!`)
- `sequence!` to use any *mutating* style functions (ending in `!`)

## Monad

Generic operations over monads are defined in the `Monad` module. To utilize
these functions, a type must implement:

- `bind` to use any *copying* style functions (not ending in `!`)
- `bind!` to use any *mutating* style functions (ending in `!`)

## Instances

In addition to interfaces and generic functions, this library also defines
default implementations of the typeclass interfaces for some of Carp's core
library types:

- `Maybe`
