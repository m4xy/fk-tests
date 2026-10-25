# fk-tests

Tests for [fckt](https://github.com/m4xy/fckt): functions marked `@Test`, each run on its own, so
one that fails doesn't stop the others.

```kotlin
// test/prices.fk
import uk.m4xy.fktests.api.Test
import pricing.Prices

@Test sumsItems() = assert(Prices.total([1, 2, 3]) == 6)
@Test emptyIsZero() = assertEquals(0, Prices.total([]))
```

A [kiln](https://github.com/m4xy/kiln) project runs them with fk-tests' plugin, whose `fkTests()`
is a task part of `check`, so of `kiln build`:

```kotlin
// build.fk
import uk.m4xy.fktests.kiln.FkTests.*

plugins = [library("uk.m4xy:fk-tests-kiln:0.1.0")]

project = Project(name = "shop")

tasks = standardTasks + [fkTests()]
```

```
$ kiln build
> Task :compile
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

Three libraries, each a package of its own under `uk.m4xy.fktests`:

```
fk-tests          uk.m4xy.fktests.api        What tests are written with: Test. The tests see it, and nothing else does.

fk-tests-runner   uk.m4xy.fktests.runner     What runs them. Only the program made from the tests uses it.
                  ├── Runner                 What fckt's main calls: test, suite and plan make the plan; main runs it
                  ├── plan                   What's to run: Plan, Suite, TestCase, TestId, Location
                  ├── results                How it went: TestResult, its Outcome, a Failure, a Summary
                  ├── events                 TestEvents, what a run tells whoever's listening
                  ├── execution              Execution.run, each test on its own, as a Selection chooses
                  └── reporting              Handlers of TestEvents: ConsoleReport, LineReport

fk-tests-kiln     uk.m4xy.fktests.kiln       The kiln plugin: fkTests(), the task that compiles and runs them
                  └── events                 Reading LineReport's lines: RunnerLine, LineReading
```

Each package uses only those above it in the list: `execution` uses `plan`, `results` and
`events`; `reporting` uses `events` and what they carry; `Runner` puts them together.

The task compiles the tests with `fckt --test test --marker uk.m4xy.fktests.api.Test --runner
uk.m4xy.fktests.runner.Runner`: fckt finds the functions marked `@Test`, and makes a main that
has `Runner` make a plan of them and run it. fckt and kiln know neither library; `fkTests(uses =
..., runner = TestRunner(...))` takes others.

`Execution` runs each test on its own, to a `TestResult`: passed, failed with a `Failure`, or
skipped, as the `Selection` from the arguments chooses; then a `Summary`. It doesn't print: it
tells `TestEvents`, as each suite and test starts and finishes, and when the run's done. A handler
reports them: `ConsoleReport`, for people; `LineReport`, an event a line, with `--report lines`,
which the plugin reads, to show them under its task; or one a test makes, as the runner's own
tests do, to see what a run did.

## Building

Each project builds with kiln, its tests run by the plugin and runner as they are here:
`kiln build` in each; `kiln publishLocal` in `runner`, here, then `kiln` publishes them to
`~/.kiln/repository`, for projects to use.
