---
title: Thoughts on Backpack, modules, and records
date: 2021-01-31
draft: false
---

In ["Implementations of the Handle pattern"](/implementations-of-the-handle-pattern), I have explored how Backpack might be used for the Handle pattern. It helped me to take a better look at Backpack and reflect a bit on modules and records in Haskell.

### What's wrong with Backpack?

Backpack works as expected but there are downsides of the current implementation:
  1. It's `cabal`-only. It means it's hard to integrate it in any big project because developers tend to choose different build tools.
  2. It's not user-friendly — it may throw [some mysterious errors](https://twitter.com/ak3n/status/1345358277803180034) without explaining what's going on. Or [parse errors in mixins block](https://github.com/haskell/cabal/issues/5150).
  3. It's not maintained. `git blame` tells that there was not much activity in Backpack modules in `cabal` repository for three years.
  4. It does not support mutual recursive modules, sealing, higher-order units, hierarchical modules, and other things. The functionality can be improved.
  5. It lacks documentation.
  6. It's [not supported by Nix](https://github.com/NixOS/nixpkgs/issues/40128). I don't think it's Backpack's problem but since Nix has become a very popular choice for Haskell's build infrastructure it's a strong blocker for Backpack's integration.
  7. It's not used. Even if we close the eyes on problems with cabal or nix, the developers don't use Backpack because it's too [heavy-weight to use](https://www.reddit.com/r/haskell/comments/7ea0qg/backpack_why_does_this_cabal_file_fail_to_build/dq46myo/) and it's unidiomatic.

These issues seem to be related: no users => bad support and bad support => no users. Backpack was an attempt to bring a subset of ML modules system to Haskell, the language with anti-modular features and a big amount of legacy code written in that style. In my opinion, that attempt failed. The ML modules system is about explicit manipulation of modules in the program by design — the style that most Haskell programmers seem to dislike.

What's the point of having an unused feature which is unmaintained and unfinished then? I write this not to criticize Backpack, but to understand its future. It's a great project that explored the important design space. The aforementioned downsides can be fixed and improved. The real question is "Should the community invest resources in that?"

### What's in the proposals

I looked at [the GHC proposals](https://github.com/ghc-proposals/ghc-proposals/) to see if there are any improvements of modularity in the future and, in my opinion, most proposals are focused on concrete problems instead of reflecting on the whole language. It may work well but also it may create inconsistency in the language.

#### [QualifiedDo](https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0216-qualified-do.rst)

`QualifiedDo` brings syntax sugar for overloading `do` notation:

```haskell
{-# LANGUAGE LinearTypes #-}
{-# LANGUAGE NoImplicitPrelude #-}
module Control.Monad.Linear (Monad(..)) where

class Monad m where
  return :: a #-> m a
  (>>=) :: m a #-> (a #-> m b) #-> mb

-----------------

module M where

import qualified Control.Monad.Linear as Linear

f :: Linear.Monad m => a #-> m b
f a = Linear.do
  b <- someLinearFunction a Linear.>>= someOtherLinearFunction
  c <- anotherLinearFunction b
  Linear.return c

g :: Monad m => a -> m b
g a = do
  b <- someNonLinearFunction a >>= someOtherNonLinearFunction
  c <- anotherNonLinearFunction b
  return c
```

`Linear.do` overloads `return` and `(>>=)` with functions from `Control.Monad.Linear`. Hey, isn't this is why Backpack was created? To replace the implementation? Also, it can be achieved with records. And the authors used that solution in [linear-base](https://github.com/tweag/linear-base/blob/0d6165fbd8ad84dd1574a36071f00a6137351637/src/System/IO/Resource.hs#L119-L120):

```haskell
(<*>) :: forall a b. RIO (a ->. b) ->. RIO a ->. RIO b
f <*> x = do
    f' <- f
    x' <- x
    Linear.pure $ f' x'
  where
    Linear.Builder { .. } = Linear.monadBuilder
```

The quote from the proposal:

>There is a way to emulate ``-XQualifiedDo`` in current GHC using ``-XRecordWildcards``: have no ``(>>=)`` and such in scope, and import a builder with ``Builder {..} = builder``. It is used in `linear-base`. This is not a very good solution: it is rather a impenetrable idiom, and, if a single function uses several builders, it yields syntactic contortion (which is why shadowing warnings are deactivated [here](https://github.com/tweag/linear-base/blob/0d6165fbd8ad84dd1574a36071f00a6137351637/src/System/IO/Resource.hs#L1))

I don't mind about using records. It might be uncomfortable, but not worthing to patch the language. Maybe I'm missing something important.

At first glance, this syntactic extension may even look okay, but a more fundamental solution would be local modules support where you can open a module locally and bring its content into the scope:

```haskell
(<*>) :: forall a b. RIO (a ->. b) ->. RIO a ->. RIO b
f <*> x = let open Linear in do
  f' <- f
  x' <- x
  Linear.pure $ f' x'
```

```haskell
(<*>) :: forall a b. RIO (a ->. b) ->. RIO a ->. RIO b
f <*> x = do
    f' <- f
    x' <- x
    Linear.pure $ f' x'
  where
    { .. } = open Linear
```

It's just like opening the records! It might be tedious to write these imports instead of `Linear.do`, but we wouldn't need to bring new functionality to the language if we had local imports. Maybe it means that "do notation" is really important to the Haskell community but to me, it feels like a temporary hack at the moment, not a fundamental part of the language.

#### [Local modules](https://github.com/goldfirere/ghc-proposals/blob/local-modules/proposals/0000-local-modules.rst)

`LocalModules` sounds like a good step in the right direction. It will be possible to create modules in modules and to open them locally! Just like in the example above with `Linear` module. Another good thing is that each `data` declaration implicitly creates a new local module, [like in Agda](https://agda.readthedocs.io/en/v2.6.1/language/record-types.html#record-modules). I'm not sure if it's possible to open them, I haven't found it in the proposal, but that would unify the opening of records and modules. Unfortunately, the proposal doesn't explore the interaction with Backpack:

>This proposal does not appear to interact with Backpack. It does not address signatures, the key feature in Backpack. Perhaps the ideas here could be extended to work with signatures.

Does it mean that there will be `LocalSignatures` in the future? Why not design the whole mechanism at once? Is there a risk of missing something important that would be hard to fix later?

#### [First class modules](https://github.com/michaelpj/ghc-proposals/blob/imp/first-class-modules/proposals/0000-first-class-modules.rst)

This proposal is about making a module a first-class entity in the language. Its status is *dormant* because it's too much to change while the benefits seem to be the same as in `LocalModules`. While `LocalModules` is a technical proposal that goes into details, `First class modules` is more about language design proposal. `LocalModules` does not replace `First class modules`, but *a part of it*. This is exactly what I was looking for, the proposal that tries to build a vision for the language, what it might look in the future. The proposal mentions Backpack only once:

> Interface files must be able to handle the possibility that an exported name refers to a module. This may have some interaction with Backpack.

Unfortunately, the work on this proposal was stopped because most interesting things were described in `LocalModules` — a proposal that is about *local namespaces*, not *modules*. There is nothing about abstraction there.

#### Summing up

It's important to remember that Haskell modules are just namespaces without any additional features. It's possible to import the contents of a module, everything or only specific things, and control what to export (except instances). While `LocalModules` will significantly improve developers' lives providing a flexible way to deal with scopes, it's unclear what the whole picture with modules will look like. And why Backpack is ignored by the proposals? What's wrong with it?

The Backpack is ignored because it's not a proper part of the Haskell language. It's a mechanism that consists of *mixin linking* which is indifferent to Haskell source code, and *typechecking against interfaces*, which is purely the concern of the compiler. That's why it depends so much on Cabal — a frontend client for mixins description which can be replaced the compiler supports typechecking of interfaces. More details are available in Edward Z. Yang's [thesis](https://github.com/ezyang/thesis/releases/tag/rev20170925).

### Modules and records, and type classes

We understood that Backpack is a tool that uses the compiler and the package manager to express the features that are not supported in the language internally. Is it possible for Haskell to go further and improve the modularity in the language internally? To fit together anti-modular type classes and some subset of modules with abstract types? I want to explore what's in the ML languages at first.

#### What's in the ML land

Let's take a look at what happens in languages with the ML modules system. The users of these languages accept the cost of the explicitness. Although, they would be glad to reduce the boilerplate when it's necessary.

The theoretical works demonstrate that type classes and modules are not that different. ["ML Modules and Haskell Type Classes: A Constructive Comparison"](https://github.com/ak3n/modules-papers/blob/master/pdfs/Wehr_ML_modules_and_Haskell_type_classes.pdf) showed that a subset of Haskell with type classes can be expressed with ML modules and vice versa with some limitations. ["Modular Type Classes"](https://github.com/ak3n/modules-papers/blob/master/pdfs/main-long.pdf) went further in expressing type classes with modules. It starts with modules as the fundamental concept and then recovers type classes as a particular mode of use of modularity.

The example of bringing type classes to the language with modules can be found in ["Modular implicits"](https://arxiv.org/abs/1512.01895) — an extension to the OCaml language for ad-hoc polymorphism inspired by Scala implicits and modular type classes. It's not supported by mainstream OCaml yet because the proper implementation requires a big change in the language. But here is a small example of how it might look for the `Show` type class:

```ocaml
(*
  We express `Show` as a module type. It's a module signature that
  can be instantiated with different implementations. It has an
  abstract type `t` and a function `show` that takes `t` and returns `string`.

  class Show t where
    show :: t -> String
*)
module type Show = sig
  type t
  val show : t -> string
end

(*
  This is a global function `show` with an implicit argument `S : Show`.
  It means that it takes a module that implements the module type `Show`,
  an argument `x`, and calls `S.show x` using the `show` from `S`.

  show' :: Show a => a -> String
  show' = show
*)
let show {S : Show} x = S.show x

(*
  An implementation of `Show` for `int`.

  instance Show Int where
    show = string_of_int
*)
implicit module Show_int = struct
  type t = int
  let show x = string_of_int x
end

(*
  An implementation of `Show` for `float`.

  instance Show Float where
    show = string_of_float
*)
implicit module Show_float = struct
  type t = float
  let show x = string_of_float x
end

(*
  An implementation of `Show` for `list`.
  Since `list` contains elements of some type `t` we require
  a module `S` to show them.

  instance Show a => Show [a] where
    show = string_of_list show
*)
implicit module Show_list {S : Show} = struct
  type t = S.t list
  let show x = string_of_list S.show x
end

let () =
  print_endline ("Show an int: " ^ show 5);
  print_endline ("Show a float: " ^ show 1.5);
  print_endline ("Show a list of ints: " ^ show [1; 2; 3]);
```

As we can see the languages with ML modules system can get ad-hoc programming support to some degree.

#### Why modularity?

Haskell's culture relies heavily on type classes. It's a foundation for monads, do-notation, and libraries. All instances of type classes should be unique and it's a cultural belief because the *global uniqueness of instances* is [just an expectation that isn't forced by GHC](http://blog.ezyang.com/2014/07/type-classes-confluence-coherence-global-uniqueness/). What's the point of living in an anti-modular myth that forbids the integration of possible modularity features?

>It is easy to dismiss this example as an implementation wart in GHC, and continue pretending that global uniqueness of instances holds. However, the problem with global uniqueness of instances is that they are inherently nonmodular: you might find yourself unable to compose two components because they accidentally defined the same type class instance, even though these instances are plumbed deep in the implementation details of the components. This is a big problem for Backpack, or really any module system, whose mantra of separate modular development seeks to guarantee that linking will succeed if the library writer and the application writer develop to a common signature.

The example with `QualifiedDo` shows that even one of the key features of Haskell needs modularity — monads and do-notation. That required additional patching of the language because do-notation is based on type classes instead of modules.

`LocalModules` may help to deal with scopes (it's really annoying sometimes), but they won't provide the abstraction mechanism — the key feature of a module system. Modules with abstract types allow replacing the implementations because of Reynolds’s abstraction theorem.

Type classes allow to create interfaces and implement them for different types, but they are usually global and ad-hoc. They make the project, the libraries, and the whole Hackage into one global world. Not always in practice for now, but it can be very annoying to track the import statement that brings an instance into the scope especially in a big codebase. Type classes become too expensive at some point as an abstraction tool.

#### Local signatures

I wrote `LocalSignatures` as a joke when was writing about `LocalModules`, but then later I understood that it might work since GHC already supports type checking against interfaces it can be used to implement signatures on the language level. It might require lots of syntax changes though. Something like that:

```haskell
module type EQ where
  data T
  eq :: T -> T -> Bool

module type MAP where
  data Key
  data Map a
  empty :: Map a
  lookup :: Key -> Map a -> Maybe a
  add :: Key -> a -> Map a -> Map a

module Map (Key :: EQ) as MAP with (type Key as Key.T) where
  type Key = Key.T
  type Map a = Key -> Maybe a
  empty x = Nothing
  lookup x m = m x
  add x y m = \z -> if Key.eq z x then Just y else m z
```

We have created two signatures (module types) `EQ` and `MAP`. And a module `Map` that implements `MAP`. Writing `as MAP`, we seal the module `Map` and hide the implementation details behind the signature `MAP`, specifying that `Key.T` is the same as `Key`.

This is just an example to demonstrate how it might look like. It introduces a module system to the language but on a different level. We can't pass a module to a function — modules and core terms are separated.

#### Modules and records

The Handle pattern demonstrates that Haskell's modules and records are alike in some way — they implement the Handle interface provided to the user, both containing functions. Records are dynamic and can be replaced in the runtime while signatures are static and can be specialized during the compilation. What if we could merge them into one entity?

[1ML](https://github.com/ak3n/modules-papers/blob/master/pdfs/1ml-jfp-draft.pdf) does that — it merges two languages in one: *core* with types and expressions, and *modules*, with signatures, structures and functors. And requires only System Fω. The example for local signatures mentioned above is the code I adapted from 1ML. Here is the original version:

```ocaml
type EQ =
{
  type t;
  eq : t -> t -> bool;
};

type MAP =
{
  type key;
  type map a;
  empty 'a : map a;
  lookup 'a : key -> map a -> opt a;
  add 'a : key -> a -> map a -> map a;
};

Map (Key : EQ) :> MAP with (type key = Key.t) =
{
  type key = Key.t;
  type map a = key -> opt a;
  empty = fun x => none;
  lookup x m = m x;
  add x y m = fun z => if Key.eq z x then some y else m z;
};

datasize = 117;
OrderedMap = Map {include Int; eq = curry (==)} :> MAP;
HashMap = Map {include Int; eq = curry (==)} :> MAP;

Map = if datasize <= 100 then OrderedMap else HashMap : MAP;
```

Looks similar, but now we can choose what module to use by analyzing the runtime value similar to what can be done in Haskell with records, vinyl, or something else. But they lack the abstract types support.

### Conclusions

I haven't thought about how local signatures or first-class modules may interact with type classes to achieve the same experience as with implicits in Agda, Scala, or OCaml. According to [the recent paper on new implicits calculus](https://lirias.kuleuven.be/2343745), implicits don't have global uniqueness of instances, just GHC's type classes in practice, but have coherence and stability of type substitutions. It makes me wonder why not try to drop the global uniqueness property and improve the module system instead.

The recently created [Haskell Foundation](https://haskell.foundation/en/) states that it's going to address the need for *driving adoption*. It means more developers will write Haskell, more libraries, more projects, more type classes, and more instances that are *global* for the entire Hackage. I think it's important to decide what to do with Backpack, consider the improvement of the module system, and design the language more carefully taking in mind the whole picture.
