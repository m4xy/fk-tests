# fk-tests

Tests for [fckt](https://github.com/m4xy/fckt): functions marked `@Test`, each run on its own, so
one that fails doesn't stop the others.

```kotlin
// test/prices.fk
import uk.m4xy.fktests.Test
import pricing.Prices

@Test sumsItems() = assert(Prices.total([1, 2, 3]) == 6)
@Test emptyIsZero() = assertEquals(0, Prices.total([]))
```

A project uses it for its tests only, which its programs and packages don't see:

```kotlin
// build.fk
project = Project(name = "shop", testDependencies = [library("uk.m4xy:fk-tests:0.1.0")])
```

```
$ kiln test
> Task :test
  prices.fk
    passed sumsItems
    FAILED emptyIsZero: prices.fk line 5: expected:<0> but was:<1>
  1 passed, 1 failed
BUILD FAILED: tests failed
$ kiln test -- sumsItems      # or prices, or prices.sumsItems
```

A test is a function with no parameters, marked `@Test`, in a file in `test`. A failed `assert`,
`fail(...)` or an exception fails just that test; its effects are main's, handled as main's are.
The other files there are what the tests share.

## How it works

[kiln](https://github.com/m4xy/kiln) compiles the tests with `fckt --test test --runner
uk.m4xy.fktests.Testing`: fckt finds the functions marked with the runner's package's `Test`, and
makes a main that gives each to `Testing`, which runs it, says how it went, and exits with 1 if any
failed. fckt knows no test library, so another can take its place, with `testRunner` in
`build.fk`.

## Building

`kiln build` compiles and tests it; `kiln publishLocal` publishes it to `~/.kiln/repository`, for
projects to use.
