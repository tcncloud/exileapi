# exileapi Protobuf Definitions

This repository contains the Protobuf definitions for the exileapi project.

## Buf Schema Registry

These definitions are managed using [Buf](https://buf.build/). They are automatically linted, checked for breaking changes, and pushed to the [Buf Schema Registry (BSR)](https://buf.build/tcn/exileapi) via GitHub Actions.

The module name is `buf.build/tcn/exileapi`.

## Usage

You can depend on these Protobuf definitions in your own projects using Buf. Add the following dependency to your `buf.yaml` file:

```yaml
version: v2
deps:
  - buf.build/tcn/exileapi
```

Then, run `buf mod update` to fetch the dependency.

## Development

### Linting

To lint the definitions locally:
```bash
buf lint
```

### Breaking Change Detection

To check for breaking changes against the `master` branch on the BSR:
```bash
buf breaking --against buf.build/tcn/exileapi:master
```

## GitHub Actions

A GitHub Actions workflow (`.github/workflows/buf-push.yaml`) is configured to automatically:

1.  **Lint & Check Breaking Changes:** On pushes to the `master` branch.
2.  **Push to BSR:** On pushes to the `master` branch and when tags matching `v*` are created.

This ensures that the definitions in the BSR are always up-to-date with the `master` branch and tagged releases.

## Contributing

[Add contribution guidelines here if applicable] 