# workflows

Reusable GitHub Actions workflows for the swissarmyhammer repositories.

This repository holds the shared CI in one place, so each calling repository
carries one short workflow file. `swift-ci.yaml` builds and tests a Swift
package under the org test contract: unit tests run at the root, integration
tests run as a separate, named target, and the unit job runs first.

```yaml
# .github/workflows/ci.yml in a calling repository
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
      integration-package-path: IntegrationTests
```

## What is here

| File | Use |
| --- | --- |
| `.github/workflows/swift-ci.yaml` | Build and test a Swift package: a unit job, an opt-in integration job, and DocC gates. |
| `.github/workflows/ci.yaml` | Build, test, format and lint a Rust crate with cargo-nextest, rustfmt and clippy. |
| `.github/actions/swift-test` | Run `swift test` with selectors, and fail a run that measured nothing. |

## Documentation

The org test contract, the two accepted repository shapes, and every input of
`swift-ci.yaml` and `swift-test`: [docs/swift-ci.md](docs/swift-ci.md).
