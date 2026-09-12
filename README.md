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

libflutter_engine-linux-x64-debug-<sha256>.so
libflutter_engine-linux-x64-profile-<sha256>.so
libflutter_engine-linux-x64-release-<sha256>.so
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
        "file": "libflutter_engine-linux-x64-debug-<sha256>.so",
        "sha256": "..."
      },
      "profile": {
        "file": "libflutter_engine-linux-x64-profile-<sha256>.so",
        "sha256": "..."
      },
      "release": {
        "file": "libflutter_engine-linux-x64-release-<sha256>.so",
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

Read the release-mode filename from metadata, then verify its digest:

```bash
BASE="https://github.com/fushell-poj/fushell-engine-builds/releases/download/engine-${ENGINE_REVISION}"
curl -fL -o metadata.json "$BASE/metadata.json"
FILE="$(jq -r '.artifacts.x86_64.release.file' metadata.json)"
SHA256="$(jq -r '.artifacts.x86_64.release.sha256' metadata.json)"
curl -fL -o libflutter_engine.so "$BASE/$FILE"
printf '%s  libflutter_engine.so\n' "$SHA256" | sha256sum --check
```

Read `metadata.json` instead of constructing artifact filenames: older releases may use the original unhashed names.

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

Open **Actions → Build Flutter Engine → Run workflow** on the branch containing these changes:

- Set `source_type` to `tag` and `source` to the exact Flutter SDK tag reported by `flutter --version`. A full framework commit is also accepted when it resolves unambiguously to a Flutter tag.
- Leave `publish_release=false` for an artifact-only build. The three libraries are retained as Actions artifacts for one day.
- Set `publish_release=true` to publish the consumer release. Leave `replace_existing=false` for a new Engine revision; existing releases fail preflight before compilation.
- To rebuild an already-published revision with Fontconfig, explicitly set both `publish_release=true` and `replace_existing=true`. Review the source and workflow before dispatching.

The workflow resolves `bin/internal/engine.version`, verifies the checkout, and builds `host_debug`, `host_profile` and `host_release` in parallel. All modes use the standalone embedder target (from `flutter/engine/src` after dependency setup):

```bash
./flutter/bin/et build --build-strategy=local --config host_debug \
  --gn-args=--enable-fontconfig \
  //flutter/shell/platform/embedder:flutter_engine
```

The Ubuntu runner installs `libfontconfig1-dev`. After each build, ELF inspection requires a `libfontconfig.so` dependency, imported `Fc*` functions, and all 16 Flutter Engine API exports loaded by Fushell. These checks verify linkage and the embedder interface; actual font fallback should still be checked in Fushell after downloading the new build. An older Flutter source without this GN option will fail rather than silently produce an engine without Fontconfig.

Before publishing, each library is renamed to `libflutter_engine-linux-x64-<mode>-<sha256>.so` and its exact name and SHA-256 are written into schema-1 `metadata.json`. No feature flag or consumer schema change is required.

### Existing releases and publication failures

New releases are staged as drafts and published only after all libraries and metadata have uploaded. Workflow runs are serialized to prevent this workflow from publishing concurrently through different Flutter tags for the same Engine revision.

With `replace_existing=true`, new content-addressed libraries upload first; existing libraries are never overwritten or deleted. An already-present hash name is downloaded and compared before reuse. `metadata.json` is replaced last, so old metadata still points to valid old libraries. GitHub's metadata replacement is a delete/upload operation, **not atomic**: downloads may briefly fail, and an interrupted metadata upload can leave metadata unavailable. Rerun with the explicit replacement option to recover; failed new releases may remain drafts. Coordinate with any manual publishers outside this workflow.

### Refreshing a Fushell project cache

Fushell validates its cached Engine against the repository marker (`.repository`) and metadata. Rebuilding the **same Engine revision in the same repository** does not automatically invalidate an already-valid cached library. After the new release is successfully published, close the running application and manually remove only the matching project cache directory:

```text
<your-project>/build/fushell_flutter_engine/<arch>/<revision>
```

For the current builds, `<arch>` is `x86_64` and `<revision>` is the full Engine revision, without the `engine-` prefix. Then rerun the normal Fushell build so it downloads the new metadata and libraries. This repository performs no automatic or global cache deletion. These workflow changes also do not remove any existing font bridge in Fushell; that is separate application work.

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
