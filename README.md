singletons
==========

[![Build Status](https://github.com/goldfirere/singletons/workflows/Haskell-CI/badge.svg)](https://github.com/goldfirere/singletons/actions?query=workflow%3AHaskell-CI)

This is the README file for the `singletons`, `singletons-th`, and
`singletons-base` libraries. This file contains documentation for the
definitions and functions in these libraries.

The `singletons` libraries were written by Richard Eisenberg (<rae@cs.brynmawr.edu>) and
with significant contributions by Jan Stolarek (<jan.stolarek@p.lodz.pl>) and
Ryan Scott (<ryan.gl.scott@gmail.com>). There
are two papers that describe the libraries. Original one, _Dependently typed
programming with singletons_, is available
[here](https://richarde.dev/papers/2012/singletons/paper.pdf) and will
be referenced in this documentation as the "singletons paper". A follow-up
paper, _Promoting Functions to Type Families in Haskell_, is available
[here](https://richarde.dev/papers/2014/promotion/promotion.pdf)
and will be referenced in this documentation as the
"promotion paper".

Ryan Scott (<ryan.gl.scott@gmail.com>) is the active maintainer.

Purpose of the libraries
------------------------

Broadly speaking, the `singletons` libraries define an ecosystem of
_singleton types_, which allow programmers
to use dependently typed techniques to enforce rich constraints among the types
in their programs. To that end, the three libraries serve the following roles:

* The `singletons` library is a small, foundational library that defines basic
  singleton-related types and definitions.
* The `singletons-th` library defines Template Haskell functionality that
  allows _promotion_ of term-level functions to type-level equivalents and
  _singling_ functions to dependently typed equivalents.
* The `singletons-base` library uses `singletons-th` to define promoted and
  singled functions from the `base` library, including the `Prelude`.

Besides the functionality of the libraries themselves, `singletons` differs
from `singletons-th` and `singletons-base` by aiming to be compatible with a
wider range of GHC versions. See the "Compatibility" section for further
details.

Some other introductions to the ideas in these libraries include:

* The singletons paper and promotion papers.
* [This blog series](https://blog.jle.im/entry/introduction-to-singletons-1.html),
  authored by Justin Le, which offers a tutorial for these libraries that
  assumes no knowledge of dependent types.

Compatibility
-------------

`singletons`, `singletons-th`, and `singletons-base` have different support
windows for requirements on the compiler version needed to build each library:

* `singletons` is a minimal library, and as such, it has a relatively
   wide support window. `singletons` must be built with GHC 8.0 or later.
   `singletons` aims to support cross-compilers as well (e.g., GHC's JavaScript
   backend).
* `singletons-th` and `singletons-base` require use of many bleeding-edge
  GHC language extensions, even more so than `singletons` itself. As such, it
  is difficult to maintain support for multiple GHC versions in any given
  release of either library, so they only support the latest major GHC version
  (currently GHC 9.12).

Any code that uses the singleton-generation functionality from `singletons-th`
or `singletons-base` needs to enable a long list of GHC extensions. This list
includes, but is not necessarily limited to, the following:

* `DataKinds`
* `DefaultSignatures`
* `EmptyCase`
* `ExistentialQuantification`
* `FlexibleContexts`
* `FlexibleInstances`
* `GADTs`
* `InstanceSigs`
* `KindSignatures`
* `NoCUSKs`
* `NoNamedWildCards`
* `NoStarIsType`
* `PolyKinds`
* `RankNTypes`
* `ScopedTypeVariables`
* `StandaloneDeriving`
* `StandaloneKindSignatures`
* `TemplateHaskell`
* `TypeAbstractions`
* `TypeApplications`
* `TypeFamilies`
* `TypeOperators`
* `UndecidableInstances`

Some notes on the use of `No*` extensions:

* `NoNamedWildCards` is needed since `singletons-th` will single code like
  `f _x = ...` to `sF (_sx :: Sing _x) = ...`, which crucially relies on the
  `_x` in `Sing _x` being treated as a type variable, not a wildcard.
* `NoStarIsType` is needed to use the `*` type family from the `PNum` class
  because with `StarIsType` enabled, GHC thinks `*` is a synonym for `Type`.

You may also want to consider toggling various warning flags:

* `-Wno-redundant-constraints`.
  The code that `singletons` generates uses redundant constraints, and there
  seems to be no way, without a large library redesign, to avoid this.
* `-fenable-th-splice-warnings`.
  By default, GHC does not run pattern-match coverage checker warnings on code
  inside of Template Haskell quotes. This is an extremely common thing to do
  in `singletons-th`, so you may consider opting in to these warnings.

Modules for singleton types
---------------------------

`Data.Singletons` (from `singletons`) exports all the basic singletons
definitions. Import this module if you are not using Template Haskell and wish
only to define your own singletons.

`Data.Singletons.Decide` (from `singletons`) exports type classes for propositional
equality. See the "Equality classes" section for more information.

`Data.Singletons.TH` (from `singletons-th`) exports all the definitions needed
to use the Template Haskell code to generate new singletons.
`Data.Singletons.Base.TH` (from `singletons-base`) re-exports
`Data.Singletons.TH` plus any promoted or singled definitions that are likely
to appear in TH-generated code. For instance, singling a
`deriving Eq` clause will make use of `SEq`, the singled `Eq` class, so
`Data.Singletons.TH` re-exports `SEq`.

`Prelude.Singletons` (from `singletons-base`) re-exports `Data.Singletons`
along with singleton definitions for various `Prelude` types. This module
provides promoted and singled equivalents of functions from the real `Prelude`.
Note that not all functions from original `Prelude` could be promoted or
singled.

The `singletons-base` library provides promoted and singled equivalents of
definitions found in several commonly used `base` library modules, including
(but not limited to) `Data.Bool`, `Data.Maybe`, `Data.Either`, `Data.List`,
`Data.Tuple`, and `Data.Void`. We also provide promoted and singled versions of
common type classes, including (but not limited to) `Eq`, `Ord`, `Show`,
`Enum`, and `Bounded`.

`GHC.TypeLits.Singletons` (from `singletons-base`) exports definitions for
working with `GHC.TypeLits`.

Functions to generate singletons
--------------------------------

The top-level functions used to generate promoted or singled definitions are
documented in the `Data.Singletons.TH` module in `singletons-th`. The most
common case is just calling the `singletons` function, which I'll describe here:

```haskell
singletons :: Q [Dec] -> Q [Dec]
```

This function generates singletons from the definitions given. Because
singleton generation requires promotion, this also promotes all of the
definitions given to the type level.

Usage example:

```haskell
$(singletons [d|
  data Nat = Zero | Succ Nat
  pred :: Nat -> Nat
  pred Zero = Zero
  pred (Succ n) = n
  |])
```

Definitions used to support singletons
--------------------------------------

This section contains a brief overview of some of the most important types
from `Data.Singletons` (from `singletons`). Please refer to the singletons
paper for a more in-depth explanation of these definitions. Many of the
definitions were developed in tandem with Iavor Diatchki.

```haskell
type Sing :: k -> Type
type family Sing
```

The type family of singleton types. A new instance of this type family is
generated for every new singleton type.

```haskell
type SingI :: forall {k}. k -> Constraint
class SingI a where
  sing :: Sing a
```

A class used to pass singleton values implicitly. The `sing` method produces
an explicit singleton value.

```haskell
type SomeSing :: Type -> Type
data SomeSing k where
  SomeSing :: Sing (a :: k) -> SomeSing k
```

The `SomeSing` type wraps up an _existentially-quantified_ singleton. Note that
the type parameter `a` does not appear in the `SomeSing` type. Thus, this type
can be used when you have a singleton, but you don't know at compile time what
it will be. `SomeSing Thing` is isomorphic to `Thing`.

```haskell
type SingKind :: Type -> Constraint
class SingKind k where
  type Demote k :: *
  fromSing :: Sing (a :: k) -> Demote k
  toSing   :: Demote k -> SomeSing k
```

This class is used to convert a singleton value back to a value in the
original, unrefined ADT. The `fromSing` method converts, say, a
singleton `Nat` back to an ordinary `Nat`. The `toSing` method produces
an existentially-quantified singleton, wrapped up in a `SomeSing`.
The `Demote` associated
kind-indexed type family maps the kind `Nat` back to the type `Nat`.

```haskell
type SingInstance :: k -> Type
data SingInstance a where
  SingInstance :: SingI a => SingInstance a
singInstance :: Sing a -> SingInstance a
```

Sometimes you have an explicit singleton (a `Sing`) where you need an implicit
one (a dictionary for `SingI`). The `SingInstance` type simply wraps a `SingI`
dictionary, and the `singInstance` function produces this dictionary from an
explicit singleton. The `singInstance` function runs in constant time, using
a little magic.

In addition to `SingI`, there are also higher-order versions named `SingI1`
and `SingI2`:

```haskell
type SingI1 :: forall {k1 k2}. (k1 -> k2) -> Constraint
class (forall x. SingI x => SingI (f x)) => SingI1 f where
  liftSing :: Sing x -> Sing (f x)

type SingI2 :: forall {k1 k2 k3}. (k1 -> k2 -> k3) -> Constraint
class (forall x y. (SingI x, SingI y) => SingI (f x y)) => SingI2 f where
  liftSing2 :: Sing x -> Sing y -> Sing (f x y)
```


Equality classes
----------------

There are two different notions of equality applicable to singletons: Boolean
equality and propositional equality.

* Boolean equality is implemented in the type family `(==)` (in the `PEq`
  class) and the `(%==`) method (in the `SEq` class).
  See the `Data.Eq.Singletons` module from `singletons-base` for more
  information.

* Propositional equality is implemented through the constraint `(~)`, the type
  `(:~:)`, and the class `SDecide`. See modules `Data.Type.Equality` and
  `Data.Singletons.Decide` from `singletons` for more information.

Which one do you need? That depends on your application. Boolean equality has
the advantage that your program can take action when two types do _not_ equal,
while propositional equality has the advantage that GHC can use the equality
of types during type inference.

Instances of `SEq`, `SDecide`, `Eq`, `TestEquality`, and `TestCoercion` are
generated when `singletons-th` is used to single a data type that has a
`deriving Eq` clause.  The structure of the `SEq` and `SDecide` instances will
match that of the derived `Eq` instance, while the other instances will be
boilerplate code:

```hs
instance Eq (SExample a) where
  _ == _ = True

-- See Data.Singletons.Decide for how decideEquality and decideCoercion are
-- implemented

instance TestEquality (SExample a) where
  testEquality = decideEquality

instance TestCoercion (SExample a) where
  testEquality = decideCoercion
```

You can also generate these instances directly through functions exported from
`Data.Singletons.TH` (from `singletons-th`) and `Data.Singletons.Base.TH` (from
`singletons-base`).

`Ord` classes
-------------

Promoted and singled versions of the `Ord` class (`POrd` and `SOrd`,
respectively) are provided in the `Data.Ord.Singletons` module from
`singletons-base`.

Just like how singling a data type with a `deriving Eq` clause will generate a
boilerplate `Eq` instance, singling a data type with a `deriving Ord` clause
will generate a boilerplate `Ord` instance:

```hs
instance Ord (SExample a) where
  compare _ _ = EQ
```

`Show` classes
--------------

Promoted and singled versions of the `Show` class (`PShow` and `SShow`,
respectively) are provided in the `Text.Show.Singletons` module from
`singletons-base`. In addition, there is a `ShowSing` constraint synonym
provided in the `Data.Singletons.ShowSing` module from `singletons`:

```haskell
type ShowSing :: Type -> Constraint
type ShowSing k = (forall z. Show (Sing (z :: k)) -- Approximately
```

This facilitates the ability to write `Show` instances for `Sing` instances.

What distinguishes all of these `Show`s? Let's use the `False` constructor as
an example. If you used the `PShow Bool` instance, then the output of calling
`Show_` on `False` is `"False"`, much like the value-level `Show Bool` instance
(similarly for the `SShow Bool` instance). However, the `Show (Sing (z :: Bool))`
instance (i.e., `ShowSing Bool`) is intended for printing the value of the
_singleton_ constructor `SFalse`, so calling `show SFalse` yields `"SFalse"`.

Instance of `PShow`, `SShow`, and `Show` (for the singleton type) are generated
when `singletons` is called on a datatype that has `deriving Show`. You can also
generate these instances directly through functions exported from
`Data.Singletons.TH` (from `singletons-th`) and
`Data.Singletons.Base.TH` (from `singletons-base`).

Errors
------

The `singletons-base` library provides two different ways to handle errors:

* The `Error` type family, from `GHC.TypeLits.Singletons`:

  ```haskell
  type Error :: a -> k
  type family Error str where {}
  ```

  This is simply an empty, closed type family, which means that it will fail
  to reduce regardless of its input. The typical use case is giving it a
  `Symbol` as an argument, so that something akin to
  `Error "This is an error message"` appears in error messages.
* The `TypeError` type family, from `Data.Singletons.Base.TypeError`. This is a
  drop-in replacement for `TypeError` from `GHC.TypeLits` which can be used
  at both the type level and the value level (via the `typeError` function).

  Unlike `Error`, `TypeError` will result in an actual compile-time error
  message, which may be more desirable depending on the use case.

Pre-defined singletons
----------------------

The `singletons-base` library defines a number of singleton types and functions
by default. These include (but are not limited to):

* `Bool`
* `Maybe`
* `Either`
* `Ordering`
* `()`
* tuples up to length 7
* lists

These are all available through `Prelude.Singletons`. Functions that
operate on these singletons are available from modules such as `Data.Singletons.Bool`
and `Data.Singletons.Maybe`.

Promoting functions
-------------------

Function promotion allows to generate type-level equivalents of term-level
definitions. Almost all Haskell source constructs are supported -- see the
"Haskell constructs supported by `singletons-th`" section of this README for a
full list.

Promoted definitions are usually generated by calling the `promote` function:

```haskell
$(promote [d|
  data Nat = Zero | Succ Nat
  pred :: Nat -> Nat
  pred Zero = Zero
  pred (Succ n) = n
  |])
```

Every promoted function and data constructor definition comes with a set of
so-called _defunctionalization symbols_. These are required to represent
partial application at the type level. For more information, refer to the
"Promotion and partial application" section below.

Users also have access to `Prelude.Singletons` and related modules (e.g.,
`Data.Bool.Singletons`, `Data.Either.Singletons`, `Data.List.Singletons`,
`Data.Maybe.Singletons`, `Data.Tuple.Singletons`, etc.) in `singletons-base`.
These provide promoted versions of function found in GHC's `base` library.

Note that GHC resolves variable names in Template Haskell quotes. You cannot
then use an undefined identifier in a quote, making idioms like this not
work:

```haskell
type family Foo a where ...
$(promote [d| ... foo x ... |])
```

In this example, `foo` would be out of scope.

Refer to the promotion paper for more details on function promotion.

Promotion and partial application
---------------------------------

Promoting higher-order functions proves to be surprisingly tricky. Consider
this example:

```hs
$(promote [d|
  map :: (a -> b) -> [a] -> [b]
  map _ []     = []
  map f (x:xs) = f x : map f xs
  |])
```

A naïve attempt to promote `map` would be:

```hs
type Map :: (a -> b) -> [a] -> [b]
type family Map f xs where
  Map _ '[]    = '[]
  Map f (x:xs) = f x : Map f xs
```

While this compiles, it is much less useful than we would like. In particular,
common idioms like `Map Id xs` will not typecheck, since GHC requires that all
invocations of type families be fully saturated. That is, the use of `Id` in
`Map Id xs` is rejected since it is not applied to one argument, which the
number of arguments that `Id` was defined with. For more information on this
point, refer to the promotion paper.

Not having the ability to partially apply functions at the type level is rather
painful, so we do the next best thing: we _defunctionalize_ all promoted
functions so that we can emulate partial application. For example, if one were
to promote the `id` function:

```hs
$(promote [d|
  id :: a -> a
  id x = x
  |]
```

Then in addition to generating the promoted `Id` type family, two
defunctionalization symbols will be generated:

```hs
type IdSym0 :: a ~> a
data IdSym0 x

type IdSym1 :: a -> a
type family IdSym1 x where
  IdSym1 x = Id x
```

In general, a function that accepts N arguments generates N+1 defunctionalization
symbols when promoted.

`IdSym1` is a _fully saturated_ defunctionalization symbol and is usually only
needed when generating code through the Template Haskell machinery. `IdSym0`
is more interesting: it has the kind `a ~> a`, which has a special arrow type
`(~>)`. Defunctionalization symbols using the `(~>)` kind are type-level
constants that can be "applied" using a special `Apply` type family:

```hs
type Apply :: (a ~> b) -> a -> b
type family Apply f x
```

Every defunctionalization symbol comes with a corresponding `Apply` instance
(except for fully saturated defunctionalization symbols). For instance, here
is the `Apply` instance for `IdSym0`:

```hs
type instance Apply IdSym0 x = Id x
```

The `(~>)` kind is used when promoting higher-order functions so that partially
applied arguments can be passed to them. For instance, here is our final attempt
at promoting `map`:

```hs
type Map :: (a ~> b) -> [a] -> [b]
type family Map f xs where
  Map _ '[]    = '[]
  Map f (x:xs) = Apply f x : Map f xs
```

Now `map id xs` can be promoted to `Map IdSym0 xs`, which typechecks without issue.

## Defunctionalizing existing type families

The most common way to defunctionalize functions is by promoting them with the
Template Haskell machinery. One can also defunctionalize existing type families,
however, by using `genDefunSymbols`. For example:

```hs
type MyTypeFamily :: Nat -> Bool
type family MyTypeFamily n

$(genDefunSymbols [''MyTypeFamily])
```

This can be especially useful if `MyTypeFamily` needs to be implemented by
hand. Be aware of the following design limitations of `genDefunSymbols`:

* `genDefunSymbols` only works for type-level declarations. Namely, it only
  works when given the names of type classes, type families, type synonyms,
  or data types. Attempting to pass the name of a term level function,
  class method, data constructor, or record selector will throw an error.
* Passing the name of a data type to `genDefunSymbols` will cause its
  data constructors to be defunctionalized but _not_ its record selectors.
* Passing the name of a type class to `genDefunSymbols` will cause the
  class itself to be defunctionalized, but /not/ its associated type families
  or methods.

Note that the limitations above reflect the current design of
`genDefunSymbols`. As a result, they are subject to change in the future.

## Defunctionalization and visible dependent quantification

Unlike most other parts of `singletons-th`, which disallow visible dependent
quantification (VDQ), `genDefunSymbols` has limited support for VDQ.
Consider this example:

```hs
type MyProxy :: forall (k :: Type) -> k -> Type
type family MyProxy k (a :: k) :: Type where
  MyProxy k (a :: k) = Proxy a

$(genDefunSymbols [''MyProxy])
```

This will generate the following defunctionalization symbols:

```hs
type MyProxySym0 ::              Type  ~> k ~> Type
type MyProxySym1 :: forall (k :: Type) -> k ~> Type
type MyProxySym2 :: forall (k :: Type) -> k -> Type
```

Note that `MyProxySym0` is a bit more general than it ought to be, since
there is no dependency between the first kind (`Type`) and the second kind
(`k`). But this would require the ability to write something like this:

```hs
type MyProxySym0 :: forall (k :: Type) ~> k ~> Type
```

This currently isn't possible. So for the time being, the kind of
`MyProxySym0` will be slightly more general, which means that under rare
circumstances, you may have to provide extra type signatures if you write
code which exploits the dependency in `MyProxy`'s kind.

Classes and instances
---------------------

This is best understood by example. Let's look at a stripped down `Ord`:

```haskell
class Eq a => Ord a where
  compare :: a -> a -> Ordering
  (<)     :: a -> a -> Bool
  x < y = case x `compare` y of
            LT -> True
	    EQ -> False
	    GT -> False
```

This class gets promoted to a "kind class" thus:

```haskell
class PEq a => POrd a where
  type Compare (x :: a) (y :: a) :: Ordering
  type (<)     (x :: a) (y :: a) :: Bool
  type x < y = ... -- promoting `case` is yucky.
```

Note that default method definitions become default associated type family
instances. This works out quite nicely.

We also get this singleton class:

```haskell
class SEq a => SOrd a where
  sCompare :: forall (x :: a) (y :: a). Sing x -> Sing y -> Sing (Compare x y)
  (%<)     :: forall (x :: a) (y :: a). Sing x -> Sing y -> Sing (x < y)

  default (%<) :: forall (x :: a) (y :: a).
                  ((x < y) ~ {- RHS from (<) above -})
		=> Sing x -> Sing y -> Sing (x < y)
  x %< y = ...  -- this is a bit yucky too
```

Note that a singled class needs to use `default` signatures, because
type-checking the default body requires that the default associated type
family instance was used in the promoted class. The extra equality constraint
on the default signature asserts this fact to the type checker.

Instances work roughly similarly.

```haskell
instance Ord Bool where
  compare False False = EQ
  compare False True  = LT
  compare True  False = GT
  compare True  True  = EQ

instance POrd Bool where
  type Compare 'False 'False = 'EQ
  type Compare 'False 'True  = 'LT
  type Compare 'True  'False = 'GT
  type Compare 'True  'True  = 'EQ

instance SOrd Bool where
  sCompare SFalse SFalse = SEQ
  sCompare SFalse STrue  = SLT
  sCompare STrue  SFalse = SGT
  sCompare STrue  STrue  = SEQ
```

The only interesting bit here is the instance signature. It's not necessary
in such a simple scenario, but more complicated functions need to refer to
scoped type variables, which the instance signature can bring into scope.
The defaults all just work.

On names
--------

The `singletons-th` library has to produce new names for the new constructs it
generates. Here are some examples showing how this is done:

1. original datatype: `Nat`

   promoted kind: `Nat`

   singleton type: `SNat` (which is really a synonym for `Sing`)


2. original datatype: `/\`

   promoted kind: `/\`

   singleton type: `%/\`


3. original constructor: `Succ`

   promoted type: `'Succ` (you can use `Succ` when unambiguous)

   singleton constructor: `SSucc`

   symbols: `SuccSym0`, `SuccSym1`


4. original constructor: `:+:`

   promoted type: `':+:`

   singleton constructor: `:%+:`

   symbols: `:+:@#@$`, `:+:@#@$$`, `:+:@#@$$$`


5. original value: `pred`

   promoted type: `Pred`

   singleton value: `sPred`

   symbols: `PredSym0`, `PredSym1`


6. original value: `+`

   promoted type: `+`

   singleton value: `%+`

   symbols: `+@#@$`, `+@#@$$`, `+@#@$$$`


7. original class: `Num`

   promoted class: `PNum`

   singleton class: `SNum`


8. original class: `~>`

   promoted class: `#~>`

   singleton class: `%~>`


Special names
-------------

There are some special cases, listed below (with asterisks\* denoting special
treatment):

1. original datatype: `[]`

   promoted kind: `[]`

   singleton type\*: `SList`


2. original constructor: `[]`

   promoted type: `'[]`

   singleton constructor\*: `SNil`

   symbols\*: `NilSym0`


3. original constructor: `:`

   promoted type: `':`

   singleton constructor\*: `SCons`

   symbols: `:@#@$`, `:@#@$$`, `:@#@$$$`


4. original datatype: `(,)`

   promoted kind: `(,)`

   singleton type\*: `STuple2`


5. original constructor: `(,)`

   promoted type: `'(,)`

   singleton constructor\*: `STuple2`

   symbols\*: `Tuple2Sym0`, `Tuple2Sym1`, `Tuple2Sym2`

   All tuples (including the 0-tuple, unit) are treated similarly. Furthermore,
   due to the lack of levity polymorphism at the kind level (see
   [GHC#14180](https://gitlab.haskell.org/ghc/ghc/issues/14180)),
   unboxed tuple data types and data constructors are promoted and singled as
   if they were boxed tuples. For example, the `(#,#)` data constructor is
   promoted to `(,)`.

6. original value: `___foo`

   promoted type\*: `US___foo` ("`US`" stands for "underscore")

   singleton value\*: `___sfoo`

   symbols\*: `US___fooSym0`

   All functions that begin with leading underscores are treated similarly.

7. Any data type constructor `Rep` (regardless of where or how `Rep` is
   defined) is promoted to `Type`. This is needed to make
   `Data.Singletons.TH.CustomStar` work.

If desired, you can pick your own naming conventions by using the
`Data.Singletons.TH.Options` module in `singletons-th`. Here is an example of
how this module can be used to prefix a singled data constructor with `MyS`
instead of `S`:

```hs
import Data.Singletons.TH
import Data.Singletons.TH.Options
import Language.Haskell.TH (Name, mkName, nameBase)

$(let myPrefix :: Name -> Name
      myPrefix name = mkName ("MyS" ++ nameBase name) in

      withOptions defaultOptions{singledDataConName = myPrefix} $
      singletons [d| data T = MkT |])
```

Haskell constructs supported by `singletons-th`
-----------------------------------------------

## Full support

The following constructs are fully supported:

* variables
* tuples
* constructors
* if statements
* infix expressions and types
* fixity declarations for infix expressions and types
* `_` patterns
* aliased patterns
* lists (including list comprehensions)
* `do`-notation
* sections
* undefined
* error
* literal expressions (for `Natural`, `Symbol`, and `Char`), including
  overloaded number literals
* datatypes that store `Natural`
* unboxed tuples (which are treated as normal tuples)
* pattern guards
* case
* let
* lambda expressions
* `!` and `~` patterns (silently but successfully ignored during promotion)
* class and instance declarations
* signatures (e.g., `(x :: Maybe a)`) in expressions
* `InstanceSigs`
* `TypeData`
* `TypeAbstractions`

## Partial support

The following constructs are partially supported:

* class constraints
* scoped type variables
* `deriving`
* finite arithmetic sequences
* records
* signatures (e.g., `(x :: Maybe a)`) in patterns
* functional dependencies
* type families
* `TypeApplications`
* wildcard types
* inferred type variable binders

See the following sections for more details.

### Class constraints

`singletons-th` supports promoting and singling class constraints, but with
some caveats about the particular sorts of constraints involved. Suppose you
have a function that looks like this:

```hs
foo :: Eq a => a -> Bool
foo x = (x == x)
```

`singletons-th` will promote `foo` to the following type family:

```hs
type Foo :: a -> Bool
type family Foo x where
  Foo x = (x == x)
```

Note that the standalone kind signature for `Foo` does not mention `PEq` (the
promoted version of `Eq`) anywhere. This is because GHC does not permit
constraints in kinds, so it would be an error to give `Foo` the kind `Eq a => a
-> Bool`. Thankfully, we don't need to mention `PEq` in `Foo`'s kind anyway, as
the promoted `(==)` type family can be used independently of a `PEq`
constraint.

`singletons-th` will single `foo` to the following definition:

```hs
sFoo :: forall a (x :: a). SEq a => Sing x -> Sing (Foo x)
sFoo sX = sX %== sX
```

This time, we can (and must) mention `SEq` (the singled version of `Eq`) in the
type signature for `sFoo`.

In general, `singletons-th` only supports constraints of the form `C T_1 ...
T_n`, where `C` is a type class constructor and each `T_i` is a type that does
not mention any other type classes. `singletons-th` does not support non-class
constraints, such as equality constraint (`(~)`) or `Coercible` constraints.

`singletons-th` only supports constraints that appear in _vanilla_ types. (See
the "Rank-n types" section below for what "vanilla" means.) `singletons-th` does
not support existential constraints that appear in data constructors.

Because `singletons-th` omits constraints when promoting types to kinds, it is
possible for some promoted types to have more general kinds than what is
intended. For example, consider this example:

```hs
bar :: Alternative f => f a
bar = empty
```

The kind of `f` in the type of `bar` should be `Type -> Type`, and GHC will
infer this because of the `Alternative f` constraint. If you promote this to
a type family, however, you instead get:

```hs
type Bar :: f a
type family Bar @f @a where
  Bar = Empty
```

Now, GHC will infer that the full kind of `Bar` is `forall {k} (f :: k -> Type)
(a :: k). f a`, which is more general that the original definition! This is not
desirable, as this means that `Bar` can be called on kinds that cannot have
`PAlternative` instances.

In general, the only way to avoid this problem is to ensure that type variables
like `f` are given explicit kinds. For example, `singletons-th` will promote
this to a type family with the correct kind:

```hs
bar :: forall (f :: Type -> Type) a. Alternative f => f a
bar = empty
```

Be especially aware of this limitation is you are dealing with classes that are
parameterized over higher-kinded type variables such as `f`.

### Scoped type variables

`singletons-th` makes an effort to track scoped type variables during promotion
so that they "just work". For instance, this function:

```hs
f :: forall a. a -> a
f x = (x :: a)
```

Will be promoted to the following type family:

```hs
type F :: forall a. a -> a
type family F x where
  F @a x = (x :: a)
```

Note the use of `@a` on the left-hand side of the type family equation, which
ensures that `a` is in scope on the right-hand side. This also works for local
definitions, so this:

```hs
f :: forall a. a -> a
f x = g
  where
    g = (x :: a)
```

Will promote to the following type families:

```hs
type F :: forall a. a -> a
type family F x where
  F @a x = LetG a x

type family LetG a x where
  LetG a x = (x :: a)
```

Note that `LetG` includes both `a` and `x` as arguments to ensure that they are
in scope on the right-hand side.

Besides `forall`s in type signatures, scoped type variables can also come
from pattern signatures. For example, this will also work:

```hs
f (x :: a) = g
  where
    g = (x :: a)
```

Not all forms of scoped type variables are currently supported:

* If a type signature brings a type variable into scope over the body of a
  function with a `forall`, then any pattern signatures must consistently use
  the same type variable in argument types that mention it. For example, while
  this will work:

  ```hs
  f :: forall a. a -> a
  f (x :: a) = a -- (x :: a) mentions the same type variable `a` from the type signature
  ```

  The following will _not_ work:

  ```hs
  f' :: forall a. a -> a
  f' (x :: b) = b -- BAD: (x :: b) mentions `b` instead of `a`
  ```

  This is because `singletons-th` would attempt to promote `f'` to the
  following:

  ```hs
  type F' :: forall a. a -> a
  type family F' x where
    F' @a (x :: b) = b
  ```

  And GHC will not infer that `b` should stand for `a`.

* There is limited support for local variables that themselves bring type
  variables into scope with `forall`s in their type signatures. For example,
  this example works:

  ```hs
  f local = g
    where
      g :: forall a. a -> a
      g x = const (x :: a) local
  ```

  This is because `f` and `g` will be promoted like so:

  ```hs
  type family F local where
    F local = G local

  type family G local x where
    G local (x :: a) = Const (x :: a) local
  ```

  It is not straightforward to give `G` a standalone kind signature, and
  without a standalone kind signature, GHC will not allow writing `@a` on the
  left-hand side of `G`'s type family equation. On the other hand, we can still
  ensure that `a` is brought into scope by writing `(x :: a)` instead of `x` on
  the left-hand side.  While the `:: a` signature was not present in the
  original definition, we include it anyway during promotion to ensure that
  definitions like `G` kind-check.

  This trick only works if the scoped type variable is mentioned in one of the
  local definition's arguments, however. If the scoped type variable is only
  mentioned in the return type, then the promoted definition will not
  kind-check. This means that examples like this one will not work:

  ```hs
  konst :: a -> Maybe Bool -> a
  konst x _ = x

  f x = konst x y
    where
      y :: forall b. Maybe b
      y = Nothing :: Maybe b
  ```

### `deriving`

`singletons-th` is slightly more conservative with respect to `deriving` than
GHC is. The only classes that `singletons-th` can derive without an explicit
deriving strategy are the following stock classes:

* `Eq`
* `Ord`
* `Show`
* `Bounded`
* `Enum`
* `Functor`
* `Foldable`
* `Traversable`

To do anything more exotic, one must explicitly indicate one's intentions by
using the `DerivingStrategies` extension. `singletons-th` fully supports the
`anyclass` strategy as well as the `stock` strategy (at least, for the classes
listed above). `singletons-th` does not support the `newtype` or `via` strategies,
as there is no equivalent of `coerce` at the type level.

### Finite arithmetic sequences

`singletons-th` has partial support for arithmetic sequences (which desugar to
methods from the `Enum` class under the hood). _Finite_ sequences (e.g.,
[0..42]) are fully supported. However, _infinite_ sequences (e.g., [0..]),
which desugar to calls to `enumFromTo` or `enumFromThenTo`, are not supported,
as these would require using infinite lists at the type level.

### Records

Record selectors are promoted to top-level functions, as there is no record
syntax at the type level. Record selectors are also singled to top-level
functions because embedding records directly into singleton data constructors
can result in surprising behavior (see
[this bug report](https://github.com/goldfirere/singletons/issues/364) for more
details on this point). TH-generated code is not affected by this limitation
since `singletons-th` desugars away most uses of record syntax. On the other
hand, it is not possible to write out code like
`SIdentity { sRunIdentity = SIdentity STrue }` by hand.

Another caveat is that GHC allows defining so-called "naughty" record selectors
that mention existential type variables that do not appear in the constructor's
return type. Naughty record selectors can be used in pattern matching, but they
cannot be used as top-level functions. Here is one example of a naughty
record selector:

```hs
data Some :: (Type -> Type) -> Type where
  MkSome :: { getSome :: f a } -> Some f
```

Because `singletons-th` promotes all records to top-level functions, however,
attempting to promote `getSome` will result in an invalid definition. (It
may typecheck, but it will not behave like you would expect.) Theoretically,
`singletons-th` could refrain from promoting naughty record selectors, but this
would require detecting which type variables in a data constructor are
existentially quantified. This is very challenging in general, so we stick to
the dumb-but-predictable approach of always promoting record selectors,
regardless of whether they are naughty or not.

### Signatures in patterns

`singletons-th` can promote basic pattern signatures, such as in the following
examples:

```hs
f :: forall a. a -> a
f (x :: a) = (x :: a)

g :: forall a. a -> a
g (x :: b) = (x :: b) -- b is the same as a
```

What does /not/ work are more advanced uses of pattern signatures that take
advantage of the fact that type variables in pattern signatures can alias other
types. Here are some examples of functions that one cannot promote:

* ```hs
  h :: a -> a -> a
  h (x :: a) (_ :: b) = x
  ```

  This typechecks by virtue of the fact that `b` aliases `a`. However, the same
  trick does not work when `h` is promoted to a type family, as a type family
  would consider `a` and `b` to be distinct type variables.
* ```hs
  i :: Bool -> Bool
  i (x :: a) = x
  ```

  This typechecks by virtue of the fact that `a` aliases `Bool`. Again, this
  would not work at the type level, as a type family would consider `a` to be
  a separate type from `Bool`.

### Functional dependencies

Inference dependent on functional dependencies is unpredictably bad. The
problem is that a use of an associated type family tied to a class with
fundeps doesn't provoke the fundep to kick in. This is GHC's problem, in
the end.

### Type families

Promoting functions with types that contain type families is likely to fail due to
[GHC#12564](https://gitlab.haskell.org/ghc/ghc/issues/12564).
Note that promoting type family _declarations_ is fine
(and often desired, since that produces defunctionalization symbols for them).

### `TypeApplications`

Currently, `singletons-th` will only promote or single code that uses
`TypeApplications` syntax in one specific situation: when type applications
are used in data constructor patterns, such as in the following example:

```haskell
foo :: Maybe a -> [a]
foo (Just @a x)  = [x :: a]
foo (Nothing @a) = [] :: [a]
```

For all other forms of `TypeApplications`, `singletons-th` will simply drop any
visible type applications. For example, `id @Bool True` will be promoted to
`Id True` and singled to `sId STrue`. See
[#378](https://github.com/goldfirere/singletons/issues/378) for a discussion
of how `singletons-th` may support `TypeApplications` in more places in the
future.

On the other hand, `singletons-th` does make an effort to preserve the order of
type variables when promoting and singling certain constructors. These include:

* Kind signatures of promoted top-level functions
* Type signatures of singled top-level functions
* Kind signatures of singled data type declarations
* Type signatures of singled data constructors
* Kind signatures of singled class declarations
* Type signatures of singled class methods

For example, consider this type signature:

```haskell
const2 :: forall b a. a -> b -> a
```

The promoted version of `const` will have the following kind signature:

```haskell
type Const2 :: forall b a. a -> b -> a
```

The singled version of `const2` will have the following type signature:

```haskell
sConst2 :: forall b a (x :: a) (y :: a). Sing x -> Sing y -> Sing (Const x y)
```

Therefore, writing `const2 @T1 @T2` works just as well as writing
`Const2 @T1 @T2` or `sConst2 @T1 @T2`, since the signatures for `const2`, `Const2`,
and `sConst2` all begin with `forall b a.`, in that order. Again, it is worth
emphasizing that the TH machinery does not support promoting or singling
`const2 @T1 @T2` directly, but you can write the type applications by hand if
you so choose.

`singletons-th` also has limited support for preserving the order of type variables
for the following constructs:

* Kind signatures of defunctionalization symbols.
  The order of type variables is only guaranteed to be preserved if:

  1. The thing being defunctionalized has a standalone type (or kind)
     signature.
  2. The type (or kind) signature of the thing being defunctionalized is
     a vanilla type. (See the "Rank-n types" section above for what "vanilla"
     means.)

  If either of these conditions do not hold, `singletons-th` will fall back to
  a slightly different approach to generating defunctionalization symbols that
  does *not* guarantee the order of type variables. As an example, consider the
  following example:

  ```haskell
  data T (x :: a) :: forall b. b -> Type
  $(genDefunSymbols [''T])
  ```

  The kind of `T` is `forall a. a -> forall b. b -> Type`, which is not
  vanilla. Currently, `singletons-th` will generate the following
  defunctionalization symbols for `T`:

  ```haskell
  data TSym0 :: a ~> b ~> Type
  data TSym1 (x :: a) :: b ~> Type
  ```

  In both symbols, the kind starts with `forall a b.` rather than quantifying
  the `b` after the visible argument of kind `a`. These symbols can still be
  useful even with this flaw, so `singletons-th` permits generating them
  regardless. Be aware of this drawback if you try doing something similar
  yourself!

* Kind signatures of promoted class methods.
  The order of type variables will often "just work" by happy coincidence, but
  there are some situations where this does not happen. Consider the following
  class:

  ```haskell
  class C (b :: Type) where
    m :: forall a. a -> b -> a
  ```

  The full type of `m` is `forall b. C b => forall a. a -> b -> a`, which binds
  `b` before `a`. This order is preserved when singling `m`, but *not* when
  promoting `m`. This is because the `C` class is promoted as follows:

  ```haskell
  class PC (b :: Type) where
    type M (x :: a) (y :: b) :: a
  ```

  Due to the way GHC kind-checks associated type families, the kind of `M` is
  `forall a b. a -> b -> a`, which binds `b` *after* `a`. Moreover, the
  `StandaloneKindSignatures` extension does not provide a way to explicitly
  declare the full kind of an associated type family, so this limitation is
  not easy to work around.

  As a result, one should not depend on the order of kind variables for promoted
  class methods like `M`. The order of kind variables in defunctionalization
  symbols for promoted class methods (e.g. `MSym0` and `MSym1`) is similarly
  unspecified.

### Wildcard types

Currently, `singletons-th` can only handle promoting or singling wildcard types
if they appear within type applications in data constructor patterns, such as
in the following example:

```hs
bar :: Maybe () -> Bool
bar (Nothing @_) = False
bar (Just ())    = True
```

`singletons-th` will error if trying to promote or single a wildcard type in
any other context. Ultimately, this is due to a GHC restriction, as GHC itself
will forbid using wildcards in most kind-level contexts. For example, GHC will
permit `f :: _` but reject `type F :: _`.

## Inferred type variable binders

`singletons-th` supports promoting inferred type variable binders in most
circumstances. For example, `singletons-th` can promote this definition:

```hs
konst :: forall a {b}. a -> b -> a
konst x _ = x
```

To this type family:

```hs
type Konst :: forall a {b}. a -> b -> a
type family Konst @a x y where
  Konst @a (x :: a) (_ :: b) = x
```

There is one (somewhat obscure) corner case that `singletons-th` cannot handle,
which requires both of the following criteria to be met:

* A definition must not have any visible arguments.
* A definition must have an inferred type variable binder as the last type
  variable in an outermost `forall`.

For instance, `singletons-th` cannot promote this definition:

```hs
bad :: forall {a}. Either a Bool
bad = Right True
```

This is because `singletons-th` will attempt to generate this type family:

```hs
type Bad :: forall {a}. Either a Bool
type family Bad where
  Bad = Right True
```

GHC will not kind-check `Bad`, however. GHC will kind-check the standalone kind
signature and conclude that `Bad` has arity 0, i.e., that it does not bind any
arguments (visible or invisible). However, the definition of `Bad` requires an
arity of 1, as it implicitly binds an argument:

```hs
Bad @{a} = Right @{a} @Bool True
```

In order to make this kind-check, we would need to be able to generate something
like this:

```hs
type Bad :: forall {a}. Either a Bool
type family Bad @{a} where
  Bad = Right True
```

However, GHC does not allow users to things like `@{a}`, and this is by design.
(See [this
part](https://github.com/ghc-proposals/ghc-proposals/blob/10290a668608d608c3f6c6010be265cf7a02e1fc/proposals/0425-decl-invis-binders.rst#alternatives)
of the relevant GHC proposal about invisible binders in type declarations.) As
such, there is no way for `singletons-th` to promote this definition exactly as
written. As a workaround, you can change the `forall {a}` to `forall a`, or you
can remove the standalone kind signature.

## Support for promotion, but not singling

The following constructs are supported for promotion but not singleton generation:

* data constructors with contexts
* overlapping patterns
* `GADTs`
* instances of poly-kinded type classes
* literal patterns

See the following sections for more details.

### Data constructors with contexts

For example, the following datatype does not single:

```haskell
data T a where
  MkT :: Show a => a -> T a
```

Constructors like these do not interact well with the current design of the
`SingKind` class. But see
[this bug report](https://github.com/goldfirere/singletons/issues/150), which
proposes a redesign for `SingKind` (in a future version of GHC with certain
bugfixes) which could permit constructors with equality constraints.

### Overlapping patterns

Overlapping patterns pose a challenge for singling. Consider this
implementation of `isNothing`:

```hs
isNothing :: Maybe a -> Bool
isNothing Nothing = True
isNothing _       = False
```

If we single this in a naïve way, we would end up with something like this:

```hs
sIsNothing :: forall a (m :: Maybe a). Sing m -> Sing (IsNothing m)
sIsNothing SNothing = STrue
sIsNothing _        = SFalse
```

This won't typecheck, however. The issue is that due to the way GADT pattern
matches are typechecked in GHC, the second equation of `sIsNothing` has no way
of knowing that `m` should be equal to `Just`, even though the first equation
already matched `m` against `Nothing`. As a result, `IsNothing m` will not
reduce in the second equation, which means that the `SFalse`, the right-hand
side, will not typecheck.

Often times, one can work around the issue by "match flattening" the
definition. Match flattening works by removing problematic wildcard patterns in
favor of enumerating all possible constructors of the data type being matched
against. Here are the match-flattened versions of `isNothing` and `sIsNothing`,
for example:

```hs
isNothing :: Maybe a -> Bool
isNothing Nothing  = True
isNothing (Just _) = False

sIsNothing :: forall a (m :: Maybe a). Sing m -> Sing (IsNothing m)
sIsNothing SNothing  = STrue
sIsNothing (SJust _) = SFalse
```

Now each equation matches on a data constructor of `Maybe`, so all is well.
Note that the wildcard pattern inside of `Just _` is fine. The only time a
wildcard pattern is problematic is when it prevents a type family in the type
signature from reducing (e.g., `IsNothing`).

Be warned that match flattening is not a perfect workaround. If you have a
function with n arguments that need to be flattened, then the flattened
version will have approximately n² equations. Nevertheless, it is the only
workaround available at the moment.

Note that overlapping patterns are sometimes not obvious. For example, the
`filter` function does not single due to overlapping patterns:

```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter _pred []    = []
filter pred (x:xs)
  | pred x         = x : filter pred xs
  | otherwise      = filter pred xs
```

Overlap is caused by `otherwise` catch-all guard, which is always true and thus
overlaps with `pred x` guard. This can be workaround by either replacing the
guards with an `if pred x then ... else ...` expression or a
`case pred x of { True -> ...; False -> ...}` expression.

Another non-obvious source of overlapping patterns comes from partial pattern
matches in `do`-notation. For example:

```haskell
f :: [()]
f = do
  Just () <- [Nothing]
  return ()
```

This has overlap because the partial pattern match desugars to the following:

```haskell
f :: [()]
f = case [Nothing] of
      Just () -> return ()
      _ -> fail "Partial pattern match in do notation"
```

Here, it is more evident that the catch-all pattern `_` overlaps with the
one above it. Again, this can be worked around by rewriting the partial pattern
match to an explicit `case` expression that matches on both `Just` and
`Nothing`.

### `GADTs`

Singling GADTs is likely to fail due to the generated `SingKind` instances
not typechecking. (See
[#150](https://github.com/goldfirere/singletons/issues/150)).
However, one can often work around the issue by suppressing the generation
of `SingKind` instances by using custom `Options`. See the `T150` test case
for an example.

### Instances of poly-kinded type classes

Singling instances of poly-kinded type classes is likely to fail due to
[#358](https://github.com/goldfirere/singletons/issues/358).
However, one can often work around the issue by using `InstanceSigs`. For
instance, the following code will not single:

```haskell
class C (f :: k -> Type) where
  method :: f a

instance C [] where
  method = []
```

Adding a type signature for `method` in the `C []` is sufficient
to work around the issue, though:

```haskell
instance C [] where
  method :: [a]
  method = []
```

### Literal patterns

Patterns which match on numeric, string, or character literals cannot be
singled. Using this code as an example:

```hs
isZero :: Natural -> Bool
isZero 0 = True
isZero _ = False
```

Due to the way that the singled version of `Natural` works, this code would
need to be singled into something like this:

```hs
sIsZero :: forall (n :: Natural). Sing n -> Sing (IsZero n)
sIsZero sn =
  case sameNat sn (SNat @0) of
    Just Refl -> STrue
    Nothing   -> SFalse
```

This runs into the same issues with overlapping patterns discussed earlier.
Unfortunately, there is not a good workaround this time. It is impossible to
match-flatten `isZero` since there are infinitely many `Natural` patterns.

If you _really_ want to make something like this work, one can write
`unsafeCoerce SFalse` to force it to typecheck. The `singletons-base` library
itself uses this trick in certain places when it has no other alternatives. See
the source code for the `SEq Natural` instance for inspiration. `singletons-th`
will not generate such code for you, however, so if you want to reach for
`unsafeCoerce`, you are on your own.

## Support for singling, but not promotion

The following constructs are supported for singleton generation but not promotion:

* bang patterns

See the following sections for more details.

### Bang patterns

Bang patterns (e.g., `f !x = (x, x)`) cannot be translated to a type-level
setting as type families lack an equivalent of bang patterns. As a result,
`singletons-th` will ignore any bang patterns and will simply promote the
underyling pattern instead.

## Little to no support

The following constructs are either unsupported or almost never work:

* datatypes that store arrows or `Symbol`
* rank-n types
* embedded type expressions and patterns
* promoting `TypeRep`s
* `TypeApplications`
* Irrefutable patterns
* `{-# UNPACK #-}` pragmas
* partial application of the `(->)` type
* invisible type patterns
* `AllowAmbiguousTypes`

See the following sections for more details.

### Arrows, `Symbol`, and literals

As described in the promotion paper, automatic promotion of datatypes that
store arrows is currently impossible. So if you have a declaration such as

```haskell
$(promote [d|
  data Foo = Bar (Bool -> Maybe Bool)
  |])
```

you will quickly run into errors.

Literals are problematic because we rely on GHC's built-in support, which
currently is limited. Functions that operate on strings will not work because
type level strings are no longer considered lists of characters. Functions
working over strings can be promoted by rewriting them to use `Symbol`. Since
`Symbol` does not exist at the term level, it will only be possible to use the
promoted definition, but not the original, term-level one.

For now, one way to work around this issue is to define two variants of a data
type: one for use at the value level, and one for use at the type level.
The example below demonstrates this workaround in the context of a data type
that has a `Symbol` field:

```hs
import Data.Kind
import Data.Singletons.TH
import Data.Singletons.TH.Options
import Data.Text (Text)
import GHC.TypeLits.Singletons
import Language.Haskell.TH (Name)

-- Term-level
newtype Message = MkMessage Text
-- Type-level
newtype PMessage = PMkMessage Symbol

$(let customPromote :: Name -> Name
      customPromote n
        | n == ''Message  = ''PMessage
        | n == 'MkMessage = 'PMkMessage
        | n == ''Text     = ''Symbol
        | otherwise       = promotedDataTypeOrConName defaultOptions n

      customDefun :: Name -> Int -> Name
      customDefun n sat = defunctionalizedName defaultOptions (customPromote n) sat in

  withOptions defaultOptions{ promotedDataTypeOrConName = customPromote
                            , defunctionalizedName      = customDefun
                            } $ do
    decs1 <- genSingletons [''Message]
    decs2 <- singletons [d|
               hello :: Message
               hello = MkMessage "hello"
               |]
    return $ decs1 ++ decs2)
```

Here is breakdown of what each part of this code is doing:

* `Message` defines a data type with a field of type `Text` (from
  `Data.Text`). `PMessage` is what we wish to be the promoted counterpart to
  `Message`. The "`P`" in `PMessage` stands for "promoted", but this is naming
  convention is not strictly enforced; you may name your types however you
  choose.

  `PMessage` is identical to `Message` modulo names and the use of `Symbol`
  instead of `Text`. The choice of `Symbol` is intentional, since
  `Demote Symbol = Text`.
* `customPromote` defines a mapping from Template Haskell `Name`s to their
  promoted `Name` equivalents. We define special cases for the three special
  types in our program: `Message` (which will promote to `PMessage`),
  `MkMessage` (which will promote to `PMkMessage`), and `Text` (which will
  promote to `Symbol`). All other names will go through the default
  `promotedDataTypeOrConName` hook (from `Data.Singletons.TH.Options`).
* `customDefun` is like `customPromote`, but it handles defunctionalization
  symbols in particular (see the "Promotion and partial application" section).
  This is needed to ensure that partial applications of `MkMessage` are promoted
  to `PMkMessageSym0` rather than `MkMessageSym0`.
* We use `customPromote` and `customDefun` to override the `defaultOptions`
  for the Template Haskell machinery. This will ensure that everything in the
  last argument to `withOptions` will recognize the names `Message`,
  `MkMessage`, and `Symbol`, promoting them according to our custom rules.
* `genSingletons [''Message]` generates a `Sing` instance for `PMessage`,
  defunctionalization symbols for `PMkMessage`, etc. These are needed for the
  next part of the code.
* Finally, the `hello` function is promoted and singled using the Template
  Haskell machinery. Note that the literal `"hello"` works as both `Text` _and_
  a `Symbol`.

Besides `Text`/`Symbol`, another common use case for this technique is
higher-order functions, e.g.,

```hs
-- Term-level
newtype Function a b = MkFunction (a -> b)
-- Type-level
newtype PFunction a b = PMkFunction (a ~> b)
```

### Rank-n types

`singletons-th` does not support type signatures that have higher-rank types.
More precisely, the only types that can be promoted or singled are
_vanilla_ types,  where a vanilla function type is a type that:

1. Only uses a `forall` at the top level, if used at all. That is to say, it
   does not contain any nested or higher-rank `forall`s.

2. Only uses a context (e.g., `c => ...`) at the top level, if used at all,
   and only after the top-level `forall` if one is present. That is to say,
   it does not contain any nested or higher-rank contexts.

3. Contains no visible dependent quantification.

### Embedded type expressions and patterns

As a consequence of `singletons-th` not supporting types with visible dependent
quantification (see the "Rank-n types" section above), `singletons-th` will not
support embedded types in expressions or patterns. This means that
`singletons-th` will reject the following examples:

```hs
idv :: forall a -> a -> a
idv (type a) (x :: a) = x

x = idv (type Bool) True
```

### Promoting `TypeRep`s

The built-in Haskell promotion mechanism does not yet have a full story around
the kind `*` (the kind of types that have values). Ideally, promoting some form
of `TypeRep` would yield `*`, but the implementation of `TypeRep` would have to
be updated for this to really work out. In the meantime, users who wish to
experiment with this feature have two options:

1) The module `Data.Singletons.Base.TypeRepTYPE` (from `singletons-base`) has all the definitions possible for
making `*` the promoted version of `TypeRep`, as `TypeRep` is currently implemented.
The singleton associated with `TypeRep` has one constructor:

    ```haskell
    type instance Sing @(TYPE rep) = TypeRep
    ```

    (Recall that `type * = TYPE LiftedRep`.) Note that any datatypes that store
`TypeRep`s will not generally work as expected; the built-in promotion
mechanism will not promote `TypeRep` to `*`.

2) The module `Data.Singletons.TH.CustomStar` (from `singletons-th`) allows the programmer to define a subset
of types with which to work. See the Haddock documentation for the function
`singletonStar` for more info.

### Irrefutable patterns

`singletons-th` will ignore irrefutable patterns (e.g., `f ~(x, y) = (y, x)`)
and will simply promote or single the underlying patterns instead.
`singletons-th` cannot promote irrefutable patterns for the same reason it
cannot promote bang patterns: there is no equivalent syntax for type families.
Moreover, `singletons-th` cannot single irrefutable patterns since singled data
constructors are implemented as GADTs, as irrefutably matching on a GADT
constructor will not bring the underlying type information into scope. Since
essentially all singled code relies on using GADT type information in this way,
it cannot reasonably be combined with irrefutable patterns, which prevent this
key feature of GADT pattern matching.

### `{-# UNPACK #-}` pragmas

`singletons-th` will ignore `{-# UNPACK #-}` pragmas on the fields of a data
constructor (e.g., `data T = MkT {-# UNPACK #-} !()`). This is because
singled data types represent their argument types using existential type
variables, and any data constructor that explicitly uses existential
quantification cannot be unpacked. See
[GHC#10016](https://gitlab.haskell.org/ghc/ghc/-/issues/10016).

### Partial application of the `(->)` type

`singletons-th` can only promote `(->)` when it is applied to exactly two
arguments. Attempting to promote `(->)` to zero or one argument will result in
an error. As a consequence, it is impossible to promote instances like the
`Functor ((->) r)` instance, so `singletons-base` does not provide them.

### Invisible type patterns

`singletons-th` currently does not support invisible type patterns, such as the
use of `@t` in this example:

```hs
f :: a -> a
f @t x = x :: t
```

See [this `singletons`
issue](https://github.com/goldfirere/singletons/issues/583).

### `AllowAmbiguousTypes`

`singletons-th` does not currently support promoting or singling types with
ambiguous type variables, which require the `AllowAmbiguousTypes` language
extension to define. For instance, `singletons-th` does not support definitions
such as this one:

```hs
{-# LANGUAGE AllowAmbiguousTypes #-}
class HasName a where
  name :: String -- This type is ambiguous
```
