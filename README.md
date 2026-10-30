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
    FAILED emptyIsZero: prices.fk line 5: assertEquals(0, Prices.total([])) failed
      expected: 0
      actual:   1
  1 passed, 1 failed
BUILD FAILED: tests failed
$ kiln test -- sumsItems      # or prices.fk, prices.fk::sumsItems, or a section, prices.fk::Totals
```

A test is a function with no parameters, marked `@Test`, in a file in `test`, or in a module in
one: its section, as `Totals.sumsItems`. A failed `assert`, `fail(...)` or an exception fails just
that test; its effects are main's, handled as main's are. The other files there are what the tests
share.

A failed assertion says what it checked and how it went: an `assert` of a comparison, each side's
value, and `assertEquals`, both values, with where two long ones on a line first differ, or, if
either is on more than one, how they differ line by line, `-` the lines expected, `+` the actual:

```
    FAILED totals: prices.fk line 9: assert(Prices.total(items) == expected) failed
      Prices.total(items) is 7
      expected is 6
    FAILED builds: builds.fk line 14: assertEquals(expected, built) failed
      - expected, + actual:
        > Task :compile
      -   up to date
      + > Task :check
        BUILD SUCCESSFUL
```

`kiln test -- --without Network` skips the tests that do an effect, by its name or its full name,
and can be given again; the tests are told what they do, so a test of what's offline needs no tag.
A test that runs for longer than five minutes is stopped, and fails, wherever it is: a
`TestConfiguration`'s `timeout()` gives them longer, or `0`, as long as they take.

## Cases

A test that gives anything but `Unit` gives its cases, each a name and what it does: each runs on
its own, inside the test's fixtures, and is reported with its name, so one failing doesn't hide
the others. `Cases.cases` makes them of rows, each with its name:

```kotlin
import uk.m4xy.fktests.api.Cases.*

@Test sums() = cases([("pair", ([1, 2], 3)), ("empty", ([], 0))]) { items, total ->
  assertEquals(total, Prices.total(items))
}
```

```
    passed sums[pair]
    FAILED sums[empty]: prices.fk line 4: assertEquals(total, Prices.total(items)) failed
      expected: 0
      actual:   1
```

`kiln test -- sums` runs them all, and `prices.fk::sums` too.

## Snapshots

What a test makes can be kept, and checked against next time, rather than written into the test:
`assertSnapshot(name, actual)` fails unless `actual` is what was kept as `name`, and says how it
differs. A test that does says `@Snapshots`:

```kotlin
import uk.m4xy.fktests.api.*
import uk.m4xy.fktests.api.Snapshot.*

@Test @Snapshots request() = assertSnapshot("wire/request", Wire.requestBytes(request).decodeToString())
```

They're kept in the tests' folder, as `test/snapshots/wire/request.txt`, to be committed with them.
One that's missing is kept the first time, unless `CI` is set, where it fails, as it should have
been committed; `kiln test -- --update-snapshots` keeps each that's changed as it is now. What's
kept is the tests', so changing it runs them again. `Snapshots` is an effect, which main handles
with `SnapshotFiles`; a test of something that uses snapshots can keep them itself.

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
                  ├── plan                   What's to run: Plan, Suite, TestCase, its TestBody, TestId, Location
                  ├── results                How it went: TestResult, its Outcome, a Failure, as Explanations says it, with its Differences; a Summary, and Summaries.of
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

`Execution` runs each test on its own, or each of its cases, to a `TestResult`: passed, failed
with a `Failure`, or skipped, as the `Selection` chooses, or as it does an effect it's run without;
then a `Summary`. `Isolation` runs it with an alarm, which stops it when its time's up, and has
`Explanations` say why it failed: fckt follows a failed assertion's message with what it checked,
as fckt's README has it, which `Explanations` reads. `Runner` reads the `Selection` from the
arguments, a `Selector` each, written as pytest's are: `sums`, every test of that name;
`prices.fk`, every test in that file; `prices.fk::sums`, that one test; `prices.fk::Totals`, the
tests in that section, and `prices.fk::Totals.sums`, one of them; and its options, `--without`,
`--timeout`, `--snapshots` and `--update-snapshots`. Files are `Path`s, from the tests' folder.
fckt's main tells it what each test does, so it needs no tags, and the runner uses nothing of
what tests are written with: a test's cases are pairs, and snapshots an effect the tests see,
with its handler. It doesn't print: it
tells `TestEvents`, as each suite and test starts and finishes, and when the run's done. A handler
reports them: `ConsoleReport`, for people; `LineReport`, an event a line, with `--report lines`,
which the plugin reads, to show them under its task; or one a test makes, as the runner's own
tests do, to see what a run did.

## Building

The three are one build, as `settings.fk` lists `runner` and `kiln` with the project here, and
their build scripts use the plugin as it is here: `kiln build` here builds and tests them all, and
`kiln publishLocal` publishes them to `~/.kiln/repository`, for projects to use.
