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

A [kiln](https://github.com/m4xy/kiln) project runs them with fk-tests' plugin, `FkTests`, whose
task `test` is part of `check`, so of `kiln build`:

```kotlin
// settings.fk
settings = Settings(uses = [library("uk.m4xy:fk-tests-kiln:0.1.0")])
```

```kotlin
// build.fk
import uk.m4xy.kiln.fckt.Fckt
import uk.m4xy.fktests.kiln.*

project = Project(name = "shop")

plugins = [Fckt(), FkTests()]
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
$ kiln test -- sumsItems      # or prices.fk, prices.fk::sumsItems, or a section, prices.fk::Totals
```

A test is a function with no parameters, marked `@Test`, in a file in `test`, or in a module in
one: its section, as `Totals.sumsItems`. A failed `assert`, `fail(...)` or an exception fails just
that test; its effects are main's, handled as main's are. The other files there are what the tests
share.

## Formats

`Formats.assertReadsBack(format, value)` fails unless a `Format`, the standard library's way of
writing a value as text and reading it back, reads back what it writes: one check for any format,
as `assertReadsBack(SettingsText, settings)`. It says what went wrong as the format writes it, as
that's all that can show any format's values; `readBackProblem(format, value)` gives that, or
`None`, without failing.

## Fixtures

A test runs inside the fixtures `@Fixtures` lists on it, and on its sections: functions that each
take what they run, the first outermost, which can handle what the test does. A section's run
outside its tests' own, and each test gets them afresh:

```kotlin
import uk.m4xy.fktests.api.*

effect Ledger {
  record(line: String): Unit
  lines(): List<String>
}

inLedger<A>(test: @Ledger () -> A): A = {
  var kept = []
  Ledger(record = line -> { kept = kept + [line] }, lines = () -> kept) { test() }
}

@Ledger @Fixtures(inLedger)
module Sections {
  @Test startsEmpty() = assertEquals([], lines())        // inside inLedger
  @Test @Fixtures(opened) seesItOpened() = ...          // inside inLedger, then opened
}
```

A fixture is named on each test, or section, it's for, never applied by itself: what a test runs
inside is on it, or on the module it's in. A test that does what no fixture handles, and has no
default, doesn't compile.

## How it works

Three libraries, each a package of its own under `uk.m4xy.fktests`:

```
fk-tests          uk.m4xy.fktests.api        What tests are written with: Test, Fixtures. The tests see it, and nothing else does.

fk-tests-runner   uk.m4xy.fktests.runner     What runs them. Only the program made from the tests uses it.
                  ├── Runner                 What fckt's main calls: test, suite and plan make the plan; main runs it
                  ├── plan                   What's to run: Plan, Suite, TestCase, TestId, Location
                  ├── results                How it went: TestResult, its Outcome, a Failure, a Summary, and Summaries.of
                  ├── events                 TestEvents, what a run tells whoever's listening
                  ├── execution              Execution.run walks the plan; Selections chooses, Isolation runs a test on its own
                  └── reporting              Handlers of TestEvents: ConsoleReport, LineReport

fk-tests-kiln     uk.m4xy.fktests.kiln       The kiln plugin: FkTests, configured by a TestConfiguration; Testing compiles and runs them
                  └── events                 Reading LineReport's lines: RunnerLine, LineReading
```

Each package uses only those above it in the list: `execution` uses `plan`, `results` and
`events`, and fckt's `Clock`, from `fckt.time`, for how long tests take; `reporting` uses `events`
and what they carry; `Runner` puts them together, with a reporter handling `TestEvents`, and
`main` the system's clock. Nothing holds state: results are returned, and time and events are
effects, so a test of the runner can say what time it is and keep what it's told.

The task compiles the tests with `fckt --test test --marker uk.m4xy.fktests.api.Test --runner
uk.m4xy.fktests.runner.Runner`: fckt finds the functions marked `@Test`, and makes a main that
has `Runner` make a plan of them and run it. fckt and kiln know neither library; a module that
follows `TestConfiguration` says where else tests are, with what they're written with, and what
runs them, overriding only what it says, and `FkTests(Mine)` uses it:

```kotlin
module Mine : TestConfiguration {
  roots(): List<TestRoot> = [TestRoot(path("test")), TestRoot(path("integration"))]
}

plugins = [Fckt(), FkTests(Mine)]
```

The tests are fckt, compiled as `Fckt` compiles the project, so a project laid out as fckt's isn't
by default gives `FkTests` the module it configures `Fckt` with, too: `plugins = [Fckt(Layout),
FkTests(Mine, Layout)]`.

Tests that run something besides their own code say so, so they never pass against an old copy of
it: kiln's run the kiln its `compile` makes, so its build has them depend on that task, and count
that program among their inputs, which run them again when it changes:

```kotlin
module KilnTests : TestConfiguration {
  dependsOn(): List<TaskName> = [FcktTasks.compile]
  inputs(): List<Path> = [path("build/bin/kiln")]
}
```

`Execution` runs each test on its own, to a `TestResult`: passed, failed with a `Failure`, or
skipped, as the `Selection` chooses; then a `Summary`. `Runner` reads the `Selection` from the
arguments, a `Selector` each, written as pytest's are: `sums`, every test of that name;
`prices.fk`, every test in that file; `prices.fk::sums`, that one test; `prices.fk::Totals`, the
tests in that section, and `prices.fk::Totals.sums`, one of them. Files are `Path`s, from the
tests' folder. It doesn't print: it
tells `TestEvents`, as each suite and test starts and finishes, and when the run's done. A handler
reports them: `ConsoleReport`, for people; `LineReport`, an event a line, with `--report lines`,
which the plugin reads, to show them under its task; or one a test makes, as the runner's own
tests do, to see what a run did.

## Building

The three are one build, as `settings.fk` lists `runner` and `kiln` with the project here, and
their build scripts use the plugin as it is here: `kiln build` here builds and tests them all, and
`kiln publishLocal` publishes them to `~/.kiln/repository`, for projects to use.
