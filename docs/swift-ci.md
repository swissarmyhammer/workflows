# Swift CI

Reference for `.github/workflows/swift-ci.yaml` and
`.github/actions/swift-test`.

## The test contract

Each Swift repository obeys this contract:

- `swift test` at the root of the repository runs all the unit tests, and only
  the unit tests. A green run means that all the tests ran: the reported count
  shows what ran, and the configuration does not skip a test.
- The integration tests run as a separate target, and the command names that
  target. An integration test uses a real model, a network, a GPU, or much
  wall-clock time.
- Do not select the tests with an environment variable. Select the tests with
  test filters or with a test package. The `integration-gate-env` inputs stay
  only for the legacy callers, until those callers migrate.
- CI runs the unit job before the integration job. A fast unit failure must
  stop the run before the integration job takes the runner. `swift-ci.yaml`
  orders its two jobs with `needs:`. A workflow that a repository writes
  itself must set its own `needs:` edge. Without that edge, one integration
  job held the only self-hosted runner for 80 minutes, while the unit job
  stayed in the queue.

## The two accepted shapes

A repository obeys the contract in one of two shapes.

### Shape 1: selectors in one package

Keep the unit test targets and the integration test targets in the root
package. `test-filter` and `test-skip` control the unit job.
`integration-filter` and `integration-skip` control the integration job.

`--filter` and `--skip` both compare a regular expression with
`<test-target>.<test-case>/<test>`. The name of a test target therefore
selects that target. Put the slow tests in their own test target. Give the
target a name that starts with a prefix. The unit job then skips that prefix,
and the integration job filters on it.

This example runs the fast tests in one job, and the real-model tests in a
second job. The integration test targets of the package share the
`AcmeRealModel` prefix, so each selector names that prefix one time.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: swissarmyhammer/workflows/.github/workflows/swift-ci.yaml@main
    with:
      test-skip: AcmeRealModel
      integration-filter: AcmeRealModel
      integration-no-parallel: true
```

### Shape 2: a nested integration package

Put the integration suite in its own package, for example
`IntegrationTests/Package.swift`, which depends on the root package by a
path. Set `integration-package-path`, and the integration job builds that
package and runs `swift test --package-path` on it. `integration-filter`,
`integration-skip` and `integration-no-parallel` apply to that run.

The workflow enforces the two requirements of this shape:

- The unit job builds the integration package on EVERY run. The root build
  does not compile these tests. Without this build step, the tests can break
  between runs, and no run shows the breakage.
- The integration job runs after the unit job, through `needs:`.

```yaml
jobs:
  ci:
    uses: swissarmyhammer/workflows/.github/workflows/swift-ci.yaml@main
    with:
      integration-package-path: IntegrationTests
      integration-no-parallel: true
```

If the integration suite depends on mlx-swift, set
`integration-metallib-glob`. The workflow then finds the `default.metallib`
in the package's `.build` and copies it next to each `.xctest` bundle, under
both probe names (`default.metallib` and `mlx.metallib`), before the tests
run.

A repository that writes its own CI steps instead of calling `swift-ci.yaml`
must obey the same two requirements itself.

### A suite that starts an executable of the root package

The integration job removes the root `.build` at its start, and the nested
package build writes its products under `IntegrationTests/.build`. Thus no
product of the root package stands at `.build/debug` in that job. A test that
starts such a product as a subprocess, and reads it at
`.build/debug/<name>`, does not find it.

Set `integration-root-products` to the names of those products. The job then
runs `swift build --product <name>` at the repository root, for each name,
before the tests run.

```yaml
jobs:
  ci:
    uses: swissarmyhammer/workflows/.github/workflows/swift-ci.yaml@main
    with:
      integration-package-path: IntegrationTests
      integration-root-products: mcp-test-server
```

The unit job needs no such input. Its `swift build --build-tests` at the root
builds every product of the root manifest.

## `swift test` exits 0 when it matches nothing

`swift test` writes `warning: No matching test cases were run` and exits 0 when
the selectors match no test. A job that measured nothing must not report green.
The `swift-test` action reads that warning and fails the step. Both jobs of
`swift-ci.yaml` use the action, so a selector that names a target which was
renamed away fails the job.

## Keep the forensic files of the integration job

Set `integration-artifacts-path` to the directory that the integration suite
writes its transcripts, logs or recordings to. The integration job then uploads
that path as an artifact with the name `integration-artifacts`.

The upload runs after the tests in all conditions: after a clean run, after a
failure, after a time limit, and after a cancellation. Thus a run that hangs
for 30 minutes with no output still keeps the files that show what happened.
A run that writes no file uploads nothing, and the job stays green.

```yaml
jobs:
  ci:
    uses: swissarmyhammer/workflows/.github/workflows/swift-ci.yaml@main
    with:
      integration-package-path: IntegrationTests
      integration-artifacts-path: IntegrationTests/.build/recordings
```

## Keep the build between runs

By default each job removes `.build` and builds everything: the unit job and
the integration job each compile the root package and the nested package. On a
self-hosted runner this is the largest part of the run time. Set
`persistent-build: true` to keep the build between runs, so a normal push
builds only what changed. The default is `false`, and then nothing about the
build directories changes.

```yaml
jobs:
  ci:
    uses: swissarmyhammer/workflows/.github/workflows/swift-ci.yaml@main
    with:
      integration-package-path: IntegrationTests
      persistent-build: true
```

### Where the build lives

The build of the root package and the build of the nested package live in one
directory for each repository and each branch:

```text
<build-cache-dir>/<owner>/<repo>/<branch-slug>-<hash>/
  root/      the build of the root package
  nested/    the build of the nested package
  lock/      exists while a job uses the directory
  fingerprint, created, used
```

`build-cache-dir` is `swift-ci-build` in the tool cache of the runner
(`RUNNER_TOOL_CACHE`) unless you set it. A pull request uses the name of its
head branch, so a push and a pull request of one branch share the directory.
The hash keeps the branches `feat/a` and `feat-a` apart.

The workspace `.build` of the root package, and `<integration-package-path>/.build`
of the nested package, become symbolic links to `root/` and `nested/`. The
workflow does not pass `--scratch-path`. A link keeps every other path right,
because the steps and the tests keep their usual `.build` paths:

- the metallib step and the legacy gated step search the build with
  `find -H`, which follows the link;
- `BuiltProductLocator` and other code that reads `--test-bundle-path` derives
  the products directory from the bundle path, and it needs no `.build` name;
- the DocC steps and `integration-artifacts-path` find `.build` where they
  always did;
- the nested package, which depends on the root package by `..`, builds in
  `nested/`, as it builds in its own `.build` today.

The unit job and the integration job of one run use the same two directories.
The integration job thus rebuilds nothing that the unit job built, as long as
both jobs run on the same runner. With more than one runner in the pool, a job
on a runner with no saved build builds everything, which is correct and slower.
The tool cache belongs to one runner, so each runner keeps its own builds.

### The lock

Two runs of one repository must not use one directory at the same time. A job
takes the lock of its directory in its first step and releases it in its last
step. The release step runs also after a failure and after a cancellation.

The lock is a directory that `mkdir` makes, and `mkdir` is atomic. A job that
finds the lock held polls every 10 seconds, prints who holds it every minute,
and fails with an error when `build-lock-timeout-minutes` ends. The lock names
its owner: the runner process of the holding job. A lock whose owner process is
gone, for example after a crash of the runner, is stale. The next job removes a
stale lock and takes it.

The workflow does not use `concurrency` for this, for three reasons:

- A `concurrency` group keeps only one waiting run, and it cancels the others.
  A run that waits for a build directory must wait, not die.
- A `concurrency` group of the called workflow can deadlock with a group of the
  same name in the caller. The caller of this workflow often sets a group such
  as `ci-${{ github.ref }}` already.
- A push and a pull request of one branch have different `github.ref` values,
  and they use one directory. The lock sees the directory, and a group sees
  only its name.

### When the build is cleaned

The job cleans the persistent build, and builds everything, when one of these
is true:

- `clean-build` is `true`;
- the fingerprint changed since the last run. The fingerprint holds the output
  of `swift --version`, the SDK version, `DEVELOPER_DIR`, and the hash of each
  `Package.swift` file and each `Package.resolved` file that git tracks;
- the directory is older than `build-max-age-days` (7 days). A busy branch thus
  starts from nothing once a week.

The job also removes a directory of the same repository that no run used for
`build-max-age-days`, if no job holds its lock. A deleted branch thus does not
keep its build for ever.

### Floating dependencies and `swift package update`

The workflow does NOT run `swift package update`, for these reasons:

- With a tracked `Package.resolved`, the pin is the decision of the repository.
  `swift package update` would replace it with the newest revision, and the run
  would build something that nobody committed. A change of the pin changes the
  fingerprint, and the build is cleaned.
- Without a tracked `Package.resolved`, the checkout of each run removes the
  untracked `Package.resolved` file, so SwiftPM resolves with no pin. It then
  fetches the floating branch, and it moves a dependency whose branch has a new
  commit. SwiftPM rebuilds only the modules that depend on the changed
  checkout.

A repository that wants a new revision of a pinned dependency commits the new
`Package.resolved`, as before.

## Time limits, the log and the dependency report

Every step that resolves, builds, or generates documentation has
`timeout-minutes: build-timeout-minutes` (60). The step that runs the
integration tests has `integration-timeout-minutes` (60). A hung step then fails
at its limit. The plain `swift test` step keeps `test-timeout-minutes`. Raise a
limit for a repository whose clean build or whose integration suite takes
longer.

The `swift test` steps run under `script -q /dev/null`. `script` gives the
command a terminal, so the command writes each line at once and the job log
shows the progress while the tests run. `script` ends with the exit status of
the command, so a failed test still fails the step. The first line of the log
can start with `^D` and two backspace characters, which `script` writes when
standard input is not a terminal.

Before each build, a step reports the SwiftPM cache of the runner
(`~/Library/Caches/org.swift.swiftpm`): its size, the number of repository
clones, the age of the newest and the oldest entry, and each
`maintenance.lock` older than 60 minutes. The step then runs
`swift package resolve` for the root package, and for the nested package, and
it prints the time of each resolve. A slow run thus shows how much of its time
went to dependency clones. The step only reads the cache. It never removes a
cache.

## Inputs of `swift-ci.yaml`

All inputs are optional. A caller that gives no input builds the package and
runs `swift test` with no selectors.

| Input | Type | Default | Use |
| --- | --- | --- | --- |
| `developer-dir` | string | `""` | Path to an `Xcode.app/Contents/Developer` to export as `DEVELOPER_DIR`. |
| `test-filter` | string | `""` | `--filter` selectors for the unit test job. Separate them with spaces or newlines. |
| `test-skip` | string | `""` | `--skip` selectors for the unit test job. |
| `test-no-parallel` | boolean | `false` | Give `--no-parallel` to the unit test job. |
| `integration-filter` | string | `""` | `--filter` selectors for the integration job. Setting this runs the job. |
| `integration-skip` | string | `""` | `--skip` selectors for the integration job. Setting this runs the job. |
| `integration-no-parallel` | boolean | `false` | Give `--no-parallel` to the integration job. |
| `integration-package-path` | string | `""` | Path to a nested integration package. Setting this runs the integration job. |
| `integration-metallib-glob` | string | `""` | `find(1)` glob that finds a `default.metallib` to copy next to the `.xctest` bundles. |
| `integration-root-products` | string | `""` | Names of products of the ROOT package that the integration job builds at the repository root, before the tests run. Set this for a suite that starts one of them as a subprocess. |
| `integration-artifacts-path` | string | `""` | Path that the integration job uploads as an artifact after the tests. Setting this keeps the forensic files of a failed run. |
| `example-targets` | string | `""` | Names of more executable targets to build one by one. |
| `docc-target` | string | `""` | Name of a library target to build a DocC catalog for. |
| `docc-coverage-script` | string | `""` | Script that fails on a DocC coverage gap. |
| `integration-gate-env` | string | `""` | LEGACY. Name of an environment variable that unlocks a gated suite. |
| `integration-xctest-glob` | string | `""` | LEGACY. `find(1)` glob that finds the `.xctest` bundle. |
| `persistent-build` | boolean | `false` | Keep the build between runs on the same runner. See "Keep the build between runs". |
| `clean-build` | boolean | `false` | Only with `persistent-build`. Clean the persistent build at the start of this run. |
| `build-cache-dir` | string | `""` | Only with `persistent-build`. Directory that holds the persistent builds. Empty means `swift-ci-build` in the tool cache of the runner. |
| `build-lock-timeout-minutes` | number | `120` | Only with `persistent-build`. How long a job waits for the lock of its build directory. |
| `build-max-age-days` | number | `7` | Only with `persistent-build`. Age at which a build directory is cleaned. |
| `build-timeout-minutes` | number | `60` | Time limit of each step that resolves, builds, or generates documentation. |
| `integration-timeout-minutes` | number | `60` | Time limit of the step that runs the integration tests. |
| `test-timeout-minutes` | number | `360` | Time limit of the plain `swift test` step of the unit job. |
| `test-sample-after-minutes` | number | `0` | When above 0, a plain `swift test` step that is still running after this time records stack samples. |

Set `test-no-parallel` or `integration-no-parallel` for a test target that lets
only one test hold a resource at a time. Swift Testing starts the time limit of
a test before the test takes such a turnstile. Thus a parallel run spends the
limit on queue time, and a queued suite fails in the same way as a hang.

## The legacy integration inputs

`integration-gate-env` and `integration-xctest-glob` stay available for the
repositories that already use them. They have two limitations.

- They select the tests with an environment variable, not with a target.
- Exactly ONE `.xctest` bundle runs. The workflow reads the glob with
  `head -n 1`. If the glob finds more than one bundle, only the first bundle
  runs. The other bundles do not run, and they do not report.

Do not use these inputs for new work. Use `integration-filter`,
`integration-skip` or `integration-package-path`, because `swift test` runs
each bundle that the selectors match. The workflow stops with an error if a
caller gives `integration-gate-env` together with `integration-filter`,
`integration-skip` or `integration-package-path`.

`integration-metallib-glob` is not legacy. It works with
`integration-package-path`, where the glob searches the nested package's
`.build`, and it also works on the legacy `integration-gate-env` path, where
the glob searches the root `.build`.

## The `swift-test` action

Use this action directly if your repository writes its own CI steps instead of
calling `swift-ci.yaml`.

```yaml
      - uses: swissarmyhammer/workflows/.github/actions/swift-test@main
        with:
          filter: AcmeRealModel
          skip: AcmeFullDataset
          no-parallel: true
```

| Input | Default | Use |
| --- | --- | --- |
| `filter` | `""` | `--filter` selectors, separated by spaces or newlines. |
| `skip` | `""` | `--skip` selectors, separated by spaces or newlines. |
| `no-parallel` | `"false"` | Set to `true` to give `--no-parallel`. |
| `package-path` | `""` | Path given as `--package-path`. Leave empty to test the package in the working directory. |
