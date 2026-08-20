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

## `swift test` exits 0 when it matches nothing

`swift test` writes `warning: No matching test cases were run` and exits 0 when
the selectors match no test. A job that measured nothing must not report green.
The `swift-test` action reads that warning and fails the step. Both jobs of
`swift-ci.yaml` use the action, so a selector that names a target which was
renamed away fails the job.

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
| `example-targets` | string | `""` | Names of more executable targets to build one by one. |
| `docc-target` | string | `""` | Name of a library target to build a DocC catalog for. |
| `docc-coverage-script` | string | `""` | Script that fails on a DocC coverage gap. |
| `integration-gate-env` | string | `""` | LEGACY. Name of an environment variable that unlocks a gated suite. |
| `integration-xctest-glob` | string | `""` | LEGACY. `find(1)` glob that finds the `.xctest` bundle. |

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
