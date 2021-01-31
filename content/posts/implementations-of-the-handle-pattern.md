---
title: Implementations of the Handle pattern
date: 2021-01-31
draft: false
---

In ["Monad Transformers and Effects with Backpack"](https://blog.ocharles.org.uk/posts/2020-12-23-monad-transformers-and-effects-with-backpack.html), [@acid2](https://twitter.com/acid2) presented how to apply Backpack to monad transformers. There is a less-popular approach to deal with effects — [Handle](https://jaspervdj.be/posts/2018-03-08-handle-pattern.html) ([Service](https://www.schoolofhaskell.com/user/meiersi/the-service-pattern)) pattern. I recommend reading both posts at first since they answer many questions regarding the design decisions behind the Handle pattern (why `IO`, why not type classes, etc). In this post, I want to show different implementations of the Handle pattern and compare them. All examples described below are available [in this repository](https://github.com/ak3n/handle-examples).

### When you might need the Handle pattern [[simple](https://github.com/ak3n/handle-examples/tree/main/simple)]

Suppose we have a domain logic with a side effect:

```haskell
module WeatherReporter where

import qualified WeatherProvider

type WeatherReport = String

-- | Domain logic. Usually some pure code that might use mtl, free monads, etc.
createWeatherReport :: WeatherProvider.WeatherData -> WeatherReport
createWeatherReport (WeatherProvider.WeatherData temp) =
  "The current temperature in London is " ++ (show temp)

-- | Domain logic that uses external dependency to get data and process it.
getCurrentWeatherReportInLondon :: IO WeatherReport
getCurrentWeatherReportInLondon = do
  weatherData <- WeatherProvider.getWeatherData "London" "now"
  return $ createWeatherReport weatherData
```

```haskell
module WeatherProvider where

type Temperature = Int
data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

-- | This is some concrete implementation.
-- In this example we return a constant value.
getWeatherData :: Location -> Day -> IO WeatherData
getWeatherData _ _ = return $ WeatherData 30
```

At some point in time, there appeared a need for tests to ensure that the domain logic is correct. There are different ways to do that:

- integration tests
- stub implementation of the service
- minimize the logic with side effects moving as much as possible to pure functions for proper unit testing
- maybe something else

All solutions have their pros and cons and the final choice depends on many factors — especially on the number of side effects and how they interact with each other. We are interested in how to achieve the second one with the Handle pattern.

### Simple Handle [[simple-handle](https://github.com/ak3n/handle-examples/tree/main/simple-handle)]

Let's start with a simple Handle that doesn't support multiple implementations.
Here is the updated domain logic:

```haskell
module WeatherReporter where

import qualified WeatherProvider

type WeatherReport = String

-- | We hide dependencies in the handle
data Handle = Handle { weatherProvider :: WeatherProvider.Handle }

-- | Constructor for Handle
new :: WeatherProvider.Handle -> Handle
new = Handle

-- | Domain logic. Usually some pure code that might use mtl, free monads, etc.
createWeatherReport :: WeatherProvider.WeatherData -> WeatherReport
createWeatherReport (WeatherProvider.WeatherData temp) =
  "The current temperature in London is " ++ (show temp)

-- | Domain logic that uses external dependency to get data and process it.
getCurrentWeatherReportInLondon :: Handle -> IO WeatherReport
getCurrentWeatherReportInLondon (Handle wph) = do
  weatherData <- WeatherProvider.getWeatherData wph "London" "now"
  return $ createWeatherReport weatherData

```

And the implementation of the `WeatherProvider`:

```haskell
module WeatherProvider where

type Temperature = Int
data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

-- | Our Handle is empty, but usually other dependencies are stored here
data Handle = Handle

-- | Constructor for Handle
new :: Handle
new = Handle

-- | This is some concrete implementation.
-- In this example we return a constant value.
getWeatherData :: Handle -> Location -> Day -> IO WeatherData
getWeatherData _ _ _ = return $ WeatherData 30
```

We have wrapped our service with the Handle interface. It's not possible to have multiple implementations yet, but we got an interface of the service and can hide all the dependencies of the service into Handle.

### Handle with records [[records-handle](https://github.com/ak3n/handle-examples/tree/main/records-handle)]

The approach with records is described in the aforementioned posts. Records are used as a dictionary with functions just like dictionary passing with type classes but explicitly. `WeatherReporter` module stays the same — it continues to use `WeatherProvider.Handle` while the `WeatherProvider` becomes an interface:

```haskell
module WeatherProvider where

type Temperature = Int
data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

-- | The interface of `WeatherProvider` with available methods.
data Handle = Handle { getWeatherData :: Location -> Day -> IO WeatherData }
```

The good thing is that we do not need to change our domain logic at all since `getWeatherData :: Handle -> Location -> Day -> IO WeatherData` has the same type. The interface allows us to create a concrete implementation for the application:

```haskell
module SuperWeatherProvider where

import WeatherProvider

new :: Handle
new = Handle { getWeatherData = getSuperWeatherData }

-- | This is some concrete implementation `WeatherProvider` interface
getSuperWeatherData :: Location -> Day -> IO WeatherData
getSuperWeatherData _ _ = return $ WeatherData 30
```

And the stub for testing that we can control:

```haskell
module TestWeatherProvider where

import WeatherProvider

-- | This is a configuration that allows to setup the provider for tests.
data Config = Config { initTemperature :: Temperature }

new :: Config -> Handle
new config = Handle { getWeatherData = getTestWeatherData $ initTemperature config }

-- | This is an implementation `WeatherProvider` interface for tests
getTestWeatherData :: Int -> Location -> Day -> IO WeatherData
getTestWeatherData temp _ _ = return $ WeatherData temp
```

The downside of this approach is the cost of an unknown function call mentioned in ["The Service Pattern"](https://www.schoolofhaskell.com/user/meiersi/the-service-pattern) post:

> In terms of call overhead, we pay the cost of an unknown function call, which is probably a bit slower than a virtual method invocation in an OOP langauge like Java. If this becomes a performance bottleneck, we will have to avoid the abstraction and specialize at compile time. Backpack will allow us to do this in a principled fashion without losing modularity.

Here is the STG of `WeatherReporter`:

```
weatherProvider =
    \r [ds_s1C4] case ds_s1C4 of { Handle ds1_s1C6 -> ds1_s1C6; };

new = \r [eta_B1] Handle [eta_B1];

createWeatherReport1 = "The current temperature in London is "#;

$wcreateWeatherReport =
    \r [ww_s1C7]
        let {
          sat_s1Cd =
              \u []
                  case ww_s1C7 of {
                    I# ww3_s1C9 ->
                        case $wshowSignedInt 0# ww3_s1C9 [] of {
                          (#,#) ww5_s1Cb ww6_s1Cc -> : [ww5_s1Cb ww6_s1Cc];
                        };
                  };
        } in  unpackAppendCString# createWeatherReport1 sat_s1Cd;

createWeatherReport =
    \r [w_s1Ce]
        case w_s1Ce of {
          WeatherData ww1_s1Cg -> $wcreateWeatherReport ww1_s1Cg;
        };

getCurrentWeatherReportInLondon5 = "London"#;

getCurrentWeatherReportInLondon4 =
    \u [] unpackCString# getCurrentWeatherReportInLondon5;

getCurrentWeatherReportInLondon3 = "now"#;

getCurrentWeatherReportInLondon2 =
    \u [] unpackCString# getCurrentWeatherReportInLondon3;

getCurrentWeatherReportInLondon1 =
    \r [ds_s1Ch void_0E]
        case ds_s1Ch of {
          Handle wph_s1Ck ->
              case wph_s1Ck of {
                Handle ds1_s1Cm ->
                    case
                        ds1_s1Cm
                            getCurrentWeatherReportInLondon4
                            getCurrentWeatherReportInLondon2
                            void#
                    of
                    { Unit# ipv1_s1Cp ->
                          let { sat_s1Cq = \u [] createWeatherReport ipv1_s1Cp;
                          } in  Unit# [sat_s1Cq];
                    };
              };
        };

getCurrentWeatherReportInLondon =
    \r [eta_B2 void_0E] getCurrentWeatherReportInLondon1 eta_B2 void#;

Handle = \r [eta_B1] Handle [eta_B1];
```

`getCurrentWeatherReportInLondon` takes two arguments — the first one is `WeatherReporter`'s Handle dictionary which we pass to `getCurrentWeatherReportInLondon1`. We match on this dictionary to get `wph_s1Ck` — this is our `WeatherProvider`'s Handle. Matching on it we get `ds1_s1Cm` — `getWeatherData` function which is called with arguments: `getCurrentWeatherReportInLondon4 = "London"` and `getCurrentWeatherReportInLondon2 = "now"`.

The result of `getWeatherData "London" "now" = ipv1_s1Cp` is then passed to `createWeatherReport` where we show the result in `$wcreateWeatherReport` and append it to `createWeatherReport1 = "The current temperature in London is "`.

The STG looks as expected. There are two allocations: one in `getCurrentWeatherReportInLondon1` and the other one in `$wcreateWeatherReport`.

### Handle with Backpack [[backpack-handle](https://github.com/ak3n/handle-examples/tree/main/backpack-handle)]

`WeatherProvider` becomes a signature in the cabal file:

```
library domain
  hs-source-dirs: domain
  signatures:      WeatherProvider
  exposed-modules: WeatherReporter
  default-language: Haskell2010
  build-depends:    base
```

and we rename `WeatherProvider.hs` to `WeatherProvider.hsig` with a little change — instead of using a concrete type for `Temperature` we make it abstract and will instantiate in implementations.

```haskell
signature WeatherProvider where

data Temperature
instance Show Temperature

data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

data Handle

-- | The interface of `WeatherProvider` with available methods.
getWeatherData :: Handle -> Location -> Day -> IO WeatherData
```

Our implementation module is almost the same as in the simple Handle case, but we have to follow the signature and export the same types. It's possible to move all common types to a different cabal library and import in the signature and the implementations.

```haskell
module SuperWeatherProvider where

type Temperature = Int
data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

-- | Our Handle is empty, but usually other dependencies are stored here
data Handle = Handle

-- | Constructor for Handle
new :: Handle
new = Handle

-- | This is some concrete implementation.
-- In this example we return a constant value.
getWeatherData :: Handle -> Location -> Day -> IO WeatherData
getWeatherData _ _ _ = return $ WeatherData 30
```

For tests we setup a configuration type to control the behavior:

```haskell
module TestWeatherProvider where

type Temperature = Int
data WeatherData = WeatherData { temperature :: Temperature }

type Location = String
type Day = String

-- | This is a configuration that allows to setup the provider for tests.
data Config = Config { initTemperature :: Temperature }

data Handle = Handle { config :: Config }

new :: Config -> Handle
new = Handle

-- | This is an implementation `WeatherProvider` interface for tests
getWeatherData :: Handle -> Location -> Day -> IO WeatherData
getWeatherData (Handle conf) _ _ = return $ WeatherData $ initTemperature conf
```

Now we need to tell which module to use instead of `WeatherProvider` hole. There are two ways: by using mixins or by reexporting modules as `WeatherProvider` in the definition libraries:

```
library impl
  hs-source-dirs: impl
  exposed-modules: SuperWeatherProvider
  reexported-modules: SuperWeatherProvider as WeatherProvider
  default-language: Haskell2010
  build-depends: base

library test-impl
  hs-source-dirs: test-impl
  exposed-modules: TestWeatherProvider
  reexported-modules: TestWeatherProvider as WeatherProvider
  default-language: Haskell2010
  build-depends: base

executable backpack-handle-exe
  main-is: Main.hs
  build-depends: base, impl, domain
  default-language: Haskell2010

test-suite spec
  type: exitcode-stdio-1.0
  hs-source-dirs: test
  main-is: Test.hs
  default-language: Haskell2010
  build-depends: base, QuickCheck, hspec, domain, test-impl
```

That's all. We do not need to change our domain logic or tests. Here is the STG of `WeatherReporter`:

```
weatherProvider =
    \r [ds_s1Bq] case ds_s1Bq of { Handle ds1_s1Bs -> ds1_s1Bs; };

new = \r [eta_B1] Handle [eta_B1];

createWeatherReport1 = "The current temperature in London is "#;

$wcreateWeatherReport =
    \r [ww_s1Bt]
        let {
          sat_s1Bz =
              \u []
                  case ww_s1Bt of {
                    I# ww3_s1Bv ->
                        case $wshowSignedInt 0# ww3_s1Bv [] of {
                          (#,#) ww5_s1Bx ww6_s1By -> : [ww5_s1Bx ww6_s1By];
                        };
                  };
        } in  unpackAppendCString# createWeatherReport1 sat_s1Bz;

createWeatherReport =
    \r [w_s1BA]
        case w_s1BA of {
          WeatherData ww1_s1BC -> $wcreateWeatherReport ww1_s1BC;
        };

getCurrentWeatherReportInLondon3 =
    \u []
        case $wshowSignedInt 0# 30# [] of {
          (#,#) ww5_s1BE ww6_s1BF -> : [ww5_s1BE ww6_s1BF];
        };

getCurrentWeatherReportInLondon2 =
    \u []
        unpackAppendCString#
            createWeatherReport1 getCurrentWeatherReportInLondon3;

getCurrentWeatherReportInLondon1 =
    \r [ds_s1BG void_0E]
        case ds_s1BG of {
          Handle _ -> Unit# [getCurrentWeatherReportInLondon2];
        };

getCurrentWeatherReportInLondon =
    \r [eta_B2 void_0E] getCurrentWeatherReportInLondon1 eta_B2 void#;

Handle = \r [eta_B1] Handle [eta_B1];
```

We can see that GHC inlined our constant implementation in the `getCurrentWeatherReportInLondon3` — we show `30` immediately.

### Handles with Backpack [[backpack-handles](https://github.com/ak3n/handle-examples/tree/main/backpack-handles)]

I decided to go further and make `WeatherReporter` a signature as well. Turned out this step required more actions with libraries. Here is `WeatherReporter.hsig`:

```haskell
signature WeatherReporter where

import qualified WeatherProvider

type WeatherReport = String

data Handle = Handle { weatherProvider :: WeatherProvider.Handle }

-- | This is domain logic. It uses `WeatherProvider` to get the actual data.
getCurrentWeatherReportInLondon :: Handle -> IO WeatherReport
```

Then we need to split the `domain` library into two because `WeatherReport` depends on `WeatherProvider`. I tried to implement them both with one implementation library but seems it's impossible. The structure of libraries becomes the following:

```
library domain-provider
  hs-source-dirs: domain
  signatures:      WeatherProvider
  default-language: Haskell2010
  build-depends:    base

library domain-reporter
  hs-source-dirs: domain
  signatures:      WeatherReporter
  default-language: Haskell2010
  build-depends:    base, domain-provider

library impl-provider
  hs-source-dirs: impl
  exposed-modules: SuperWeatherProvider
  reexported-modules: SuperWeatherProvider as WeatherProvider
  default-language: Haskell2010
  build-depends:    base

library impl-reporter
  hs-source-dirs: impl
  exposed-modules: SuperWeatherReporter
  reexported-modules: SuperWeatherReporter as WeatherReporter
  default-language: Haskell2010
  build-depends:    base, domain-provider
```

Instead of `domain` we have `domain-provider` and `domain-reporter`. It allows to depend on them individually and instantiate with different implementations. In [the example](https://github.com/ak3n/handle-examples/blob/main/backpack-handles/backpack-handles.cabal#L52), I have instantiated the provider with implementation from `test-impl` using the reporter from `impl-reporter`. This is useful if you want to gradually write tests for different parts of the logic.

### Handle with Vinyl [[vinyl-handle](https://github.com/ak3n/handle-examples/tree/main/vinyl-handle)]

Suppose that we want to extend our `WeatherData` and return not only temperature but wind's speed too. We need to add a field to `WeatherData`:

```
data WeatherData = WeatherData { temperature :: T.Temperature, wind :: W.WindSpeed }
```

But also we want to provide separate Handles for these values: `TemperatureProvider` and `WindProvider`. We can create these two providers and then duplicate their methods in `WeatherProvider`. That might work if there are no so many methods, but what if their number will grow?

We know that records are nominally typed and can't be composed. There is a library called [vinyl](https://hackage.haskell.org/package/vinyl) that provides structural records supporting merge operation. I recommend Jon Sterling's [talk on Vinyl](https://vimeo.com/102785458) where you can learn why records are sheaves and other details on Vinyl.

Let's explore what the Handle pattern will look like if we replace records with Vinyl records. We create our data providers `TemperatureProvider` and `WindProvider`:

```haskell
module TemperatureProvider where

import HandleRec
import QueryTypes

type Temperature = Int

type Methods = '[ '("getTemperatureData", (Location -> Day -> IO Temperature)) ]

type Handle = HandleRec Methods

getTemperatureData :: Handle -> Location -> Day -> IO Temperature
getTemperatureData = getMethod @"getTemperatureData"

```

```haskell
module WindProvider where

import HandleRec
import QueryTypes

type WindSpeed = Int

type Methods = '[ '("getWindData", (Location -> Day -> IO WindSpeed)) ]

type Handle = HandleRec Methods

getWindData :: Handle -> Location -> Day -> IO WindSpeed
getWindData = getMethod @"getWindData"
```

Note that we provide `getTemperatureData` and `getWindData` functions which satisfy our Handle interface.

We can compose them together and extend them to create `WeatherProvider`:

```haskell
module WeatherProvider where

import Data.Vinyl.TypeLevel
import HandleRec
import qualified WindProvider as W
import qualified TemperatureProvider as T
import QueryTypes

data WeatherData = WeatherData { temperature :: T.Temperature, wind :: W.WindSpeed }

-- We union the methods of providers and extend it with a common method.
type Methods = '[ '("getWeatherData", (Location -> Day -> IO WeatherData))
  ] ++ W.Methods ++ T.Methods

type Handle = HandleRec Methods

getWeatherData :: Handle -> Location -> Day -> IO WeatherData
getWeatherData = getMethod @"getWeatherData"
```

Here is how our providers are implemented:

```haskell
module SuperTemperatureProvider where

import Data.Vinyl
import TemperatureProvider
import QueryTypes

new :: Handle
new = Field getSuperTemperatureData :& RNil

getSuperTemperatureData :: Location -> Day -> IO Temperature
getSuperTemperatureData _ _ = return 30
```

```haskell
module SuperWindProvider where

import Data.Vinyl
import WindProvider
import QueryTypes

new :: Handle
new = Field getSuperWindData :& RNil

getSuperWindData :: Location -> Day -> IO WindSpeed
getSuperWindData _ _ = return 5
```

```haskell
module SuperWeatherProvider where

import Data.Vinyl
import WeatherProvider
import qualified TemperatureProvider
import qualified WindProvider
import QueryTypes

new :: WindProvider.Handle -> TemperatureProvider.Handle -> Handle
new wp tp = Field getSuperWeatherData :& RNil <+> wp <+> tp

-- | This is some concrete implementation `WeatherProvider` interface
getSuperWeatherData :: Location -> Day -> IO WeatherData
getSuperWeatherData _ _ = return $ WeatherData 30 10
```

The domain logic and tests stay the same thanks to the Handle interface. The STG of `WeatherReporter`:

```
createWeatherReport2 = "The current temperature in London is "#;

createWeatherReport1 = " and wind speed is "#;

$wcreateWeatherReport =
    \r [ww_s2fz ww1_s2fA]
        let {
          sat_s2fN =
              \u []
                  case ww_s2fz of {
                    I# ww3_s2fC ->
                        case $wshowSignedInt 0# ww3_s2fC [] of {
                          (#,#) ww5_s2fE ww6_s2fF ->
                              let {
                                sat_s2fM =
                                    \s []
                                        let {
                                          sat_s2fL =
                                              \u []
                                                  case ww1_s2fA of {
                                                    I# ww9_s2fH ->
                                                        case $wshowSignedInt 0# ww9_s2fH [] of {
                                                          (#,#) ww11_s2fJ ww12_s2fK ->
                                                              : [ww11_s2fJ ww12_s2fK];
                                                        };
                                                  };
                                        } in  unpackAppendCString# createWeatherReport1 sat_s2fL;
                              } in  ++_$s++ sat_s2fM ww5_s2fE ww6_s2fF;
                        };
                  };
        } in  unpackAppendCString# createWeatherReport2 sat_s2fN;

createWeatherReport =
    \r [w_s2fO]
        case w_s2fO of {
          WeatherData ww1_s2fQ ww2_s2fR ->
              $wcreateWeatherReport ww1_s2fQ ww2_s2fR;
        };

getCurrentWeatherReportInLondon5 = "London"#;

getCurrentWeatherReportInLondon4 =
    \u [] unpackCString# getCurrentWeatherReportInLondon5;

getCurrentWeatherReportInLondon3 = "now"#;

getCurrentWeatherReportInLondon2 =
    \u [] unpackCString# getCurrentWeatherReportInLondon3;

getCurrentWeatherReportInLondon1 =
    \r [ds_s2fS void_0E]
        case ds_s2fS of {
          Handle wph_s2fV ->
              case wph_s2fV of {
                :& x1_s2fX _ ->
                    case x1_s2fX of {
                      Field _ x2_s2g1 ->
                          case
                              x2_s2g1
                                  getCurrentWeatherReportInLondon4
                                  getCurrentWeatherReportInLondon2
                                  void#
                          of
                          { Unit# ipv1_s2g4 ->
                                let { sat_s2g5 = \u [] createWeatherReport ipv1_s2g4;
                                } in  Unit# [sat_s2g5];
                          };
                    };
              };
        };

getCurrentWeatherReportInLondon =
    \r [eta_B2 void_0E] getCurrentWeatherReportInLondon1 eta_B2 void#;

Handle = \r [eta_B1] Handle [eta_B1];
```

As we can see there is not much vinyl-specific runtime overhead in this case — we pattern match on `wph_s2fV` and `x1_s2fX` to get the function `x2_s2g1`. But keep in mind that accessing an element is linear since Vinyl is based on HList and compilation time will grow because of type-level machinery. Vinyl can be replaced for an alternative with logarithmic complexity.

### Conclusions

No surprises here. Backpack works as expected specifying things at compile time. Vinyl allows to compose records and can be replaced with any [alternative](https://github.com/danidiaz/red-black-record#alternatives). The Handle pattern works since it's just a type signature  `... :: Handle -> ...`.

The Handle allows us to hide dependencies and to create interfaces, allowing us to easily replace the implementation without changes on the client-side — statically using Backpack for better performance or dynamically using records or alternatives in runtime (in first-class modules manner). Backpack might be too tedious for Handles that depend on each other but in simple cases, it introduces not much additional cost compared to records. And it's possible to mix them.

This post inspired me to write a follow-up post on [Backpack, modules, and records](/posts/thoughts-on-backpack-modules-and-records).
