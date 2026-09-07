# fushell-engine-builds

Prebuilt Flutter Engine embedder libraries for [Fushell](https://github.com/fushell-poj/fushell).

This repository builds and publishes `libflutter_engine.so` for specific Flutter Engine revisions, allowing Fushell and other embedder-based tooling to obtain the exact Engine binary required by the Flutter SDK in use without building the Flutter Engine locally.

## Why this repository exists

Flutter applications are tied to a specific Flutter Engine revision.

Using an Engine from a different Flutter release can result in incompatibilities such as mismatched Kernel binary versions or incompatible AOT artifacts.

For that reason, releases in this repository are identified by the **full Flutter Engine revision**, rather than by the Flutter semantic version alone.

For example:

```text
engine-42d3d75a56efe1a2e9902f52dc8006099c45d937
```

A consumer can obtain its required revision from:

```bash
flutter --version --machine
```

and use the returned `engineRevision` to locate the matching release.

## Release layout

Each Engine release currently contains Linux `x86_64` builds for all three Flutter runtime modes:

```text
metadata.json

libflutter_engine-linux-x64-debug.so
libflutter_engine-linux-x64-profile.so
libflutter_engine-linux-x64-release.so
```

Release tags use the following format:

```text
engine-<full-engine-revision>
```

For example:

```text
engine-42d3d75a56efe1a2e9902f52dc8006099c45d937
```

The corresponding release URL is therefore predictable from the Engine revision.

## metadata.json

Every release includes a machine-readable `metadata.json` describing the Engine build and its artifacts.

Example:

```json
{
  "schema": 1,
  "engine_revision": "42d3d75a56efe1a2e9902f52dc8006099c45d937",
  "flutter_version": "3.41.9",
  "dart_version": "3.11.5",
  "artifacts": {
    "x86_64": {
      "debug": {
        "file": "libflutter_engine-linux-x64-debug.so",
        "sha256": "..."
      },
      "profile": {
        "file": "libflutter_engine-linux-x64-profile.so",
        "sha256": "..."
      },
      "release": {
        "file": "libflutter_engine-linux-x64-release.so",
        "sha256": "..."
      }
    }
  }
}
```

Consumers should validate at least:

- `schema`
- `engine_revision`
- requested architecture and mode
- artifact SHA-256

The architecture is represented inside `artifacts` so additional architectures can be added without changing the basic metadata layout.

## Downloading an Engine

Given an Engine revision:

```bash
ENGINE_REVISION=42d3d75a56efe1a2e9902f52dc8006099c45d937
```

metadata can be downloaded with:

```bash
curl -fL \
  "https://github.com/fushell-poj/fushell-engine-builds/releases/download/engine-${ENGINE_REVISION}/metadata.json"
```

A release-mode Linux x64 Engine can then be downloaded from:

```bash
curl -fL \
  -o libflutter_engine.so \
  "https://github.com/fushell-poj/fushell-engine-builds/releases/download/engine-${ENGINE_REVISION}/libflutter_engine-linux-x64-release.so"
```

Applications are encouraged to read `metadata.json` instead of constructing artifact filenames manually, and to verify the downloaded file against its published SHA-256 digest.

## Fushell integration

Fushell resolves the Engine dynamically instead of embedding a fixed Flutter Engine into the Fushell executable.

The basic flow is:

```text
Flutter CLI
    │
    └── flutter --version --machine
                │
                ▼
          engineRevision
                │
                ▼
fushell-engine-builds release
                │
                ├── metadata.json
                │
                └── matching libflutter_engine.so
                │
                ▼
        SHA-256 verification
                │
                ▼
             cache
                │
                ▼
       application bundle
```

This allows the Fushell executable to remain independent of a particular Flutter release.

Updating the local Flutter SDK does not require rebuilding Fushell solely to update its embedded Engine. Fushell can instead resolve the Engine corresponding to the new SDK revision.

## Building

Engine builds are produced by GitHub Actions.

The workflow accepts either:

- a Flutter release tag, such as `3.41.9`
- a full 40-character Flutter framework commit

The selected Flutter source is resolved to its exact Engine revision through Flutter's:

```text
bin/internal/engine.version
```

The workflow then checks out the corresponding Flutter source and builds the embedder target in parallel for:

```text
host_debug
host_profile
host_release
```

using Flutter Engine's `et` tooling.

The resulting libraries are published as:

```text
libflutter_engine-linux-x64-debug.so
libflutter_engine-linux-x64-profile.so
libflutter_engine-linux-x64-release.so
```

Before publishing, SHA-256 digests are calculated and written into `metadata.json`.

A release is not overwritten if the same Engine revision has already been published.

## Supported platforms

Currently published:

| Platform | Architecture | Debug | Profile | Release |
| --- | --- | :---: | :---: | :---: |
| Linux | `x86_64` | ✓ | ✓ | ✓ |

Additional architectures can be represented as additional keys under `metadata.json` → `artifacts`.

## Versioning

This repository does not assign an independent version number to Flutter Engine builds.

The Flutter Engine Git revision is the canonical identifier:

```text
engine-<revision>
```

The Flutter and Dart versions stored in `metadata.json` are informational metadata describing the SDK associated with that Engine revision.

## Integrity

Published artifacts include SHA-256 digests in `metadata.json`.

Consumers should verify the digest before moving a downloaded Engine into a persistent cache or application bundle.

A recommended download flow is:

```text
download to temporary file
        │
        ▼
calculate SHA-256
        │
        ├── mismatch → delete
        │
        └── match
              │
              ▼
       promote to cache
```

## Project status

This repository is infrastructure for the Fushell project and currently focuses on the Engine configurations required by Fushell.

It is not an official Flutter project and is not affiliated with the Flutter team.
