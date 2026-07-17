---
id: "002-toolchain"
title: "The Encore build toolchain: vendored core, per-platform binaries"
status: approved
created: "2026-07-17"
implementation: pending
depends_on:
  - "000-bootstrap"
  - "001-packages-thesis"
establishes:
  - { kind: directory, path: "vendor/encore/" }
  - { kind: directory, path: "packages/toolchain/" }
  - { kind: directory, path: "packages/toolchain-darwin-arm64/" }
  - { kind: directory, path: "packages/toolchain-linux-x64/" }
  - { kind: directory, path: "packages/toolchain-linux-arm64/" }
  - { kind: file, path: ".github/workflows/publish.yml" }
  - { kind: file, path: "package.json" }
summary: >
  The first package lands: the Encore build toolchain moves from enrahitu
  to `@statecrafting/toolchain`, bringing the vendored Encore tree
  (`vendor/encore/`, MPL-2.0, pinning upstream v1.57.9) and the three
  per-platform binary packages that carry `encore-runtime.node` and
  `tsparser-encore`. It is the deepest dependency in the family: both
  enrahitu and statecraft build with it. Retires
  enrahitu/008-vendored-encore-toolchain (whose only edge is
  `vendor/encore/`) and drops `packages/` from enrahitu/018-packaged-chassis.
---

# 002: The Encore build toolchain

## 1. Purpose

The toolchain is the deepest dependency in the product family: enrahitu
builds with it, statecraft builds with it, and every stamped app builds
with it. It is named `@enrahitu/toolchain` today, which is the clearest
case of the scope lying about ownership (spec 001 section 1): statecraft
is not enrahitu, and pulls this package from npm on every install.

It moves first because everything else builds on it. Publishing
`@statecrafting/toolchain` before the other packages move means each later
migration lands against an already-published toolchain rather than a
moving target.

## 2. Territory

Four npm packages, not one. The meta package carries the drivers and
resolves a per-platform binary package at runtime:

| Package | Contents |
|---|---|
| `@statecrafting/toolchain` | the dev/build/bundle drivers; no binary |
| `@statecrafting/toolchain-darwin-arm64` | `encore-runtime.node` + `tsparser-encore` |
| `@statecrafting/toolchain-linux-x64` | same, linux x64 |
| `@statecrafting/toolchain-linux-arm64` | same, linux arm64 |

The three platform packages are `optionalDependencies` of the meta
package; npm installs only the one matching the host.

`vendor/encore/` is the upstream Encore source the binaries are built from,
pinned at `v1.57.9`. It is build-time only: the meta package's `files` list
is `bin`, `lib`, `scripts`, `README.md`, and vendored Encore appears in
none of them. It is 6.6 MB tracked; the multi-gigabyte working tree is
`target/` build output and is gitignored.

**`git archive` cannot move this tree.** enrahitu's `.gitattributes` marks
the entire Encore build tree `export-ignore`: `Cargo.toml`, `Cargo.lock`,
`proto/`, `tsparser/`, `miniredis/`, `runtimes/core/`, and
`runtimes/js/src/`. That is correct and deliberate there, because the
factory exports a stamp with `git archive` (enrahitu spec 005) and a
stamped app must not carry Encore's build tree; it consumes the published
platform packages instead. The consequence for this migration is a trap:
`git archive` silently yields 197 of 625 files, dropping every Rust source
and the workspace manifest, and the omission does not surface until
`cargo` reports a missing manifest path. The tree must be copied from the
git index by path list (`git ls-files` piped to `rsync --files-from`), not
exported.

`package.json` at the root is a build harness, not a shipped artifact: it
carries `build:runtime`, which cargo-builds `encore-js-runtime` and
`encore-tsparser` out of `vendor/encore/` and links the results into the
platform package. It is `private` and never published.

## 3. Behavior

### The MPL boundary

`vendor/encore/` is MPL-2.0 and stays MPL-2.0 (spec 001 section 3). This is
the one directory in this repository under that license. The rules that
follow from MPL-2.0 being file-level copyleft:

- Modifications to files under `vendor/encore/` are published under
  MPL-2.0. The vendored tree is not a dumping ground for our own code; a
  change there is a change to Encore, and it carries Encore's license.
- The Apache-2.0 root does not reach into it and does not need to. MPL
  files sit lawfully under a differently-licensed larger work
  (MPL-2.0 section 3.3).
- The binaries built from it (`encore-runtime.node`, `tsparser-encore`) are
  MPL-2.0 object code and ship inside the platform packages. Those packages
  therefore declare MPL-2.0 in their manifests, not Apache-2.0: a package
  whose entire payload is built from MPL sources does not get to call
  itself permissive. This is a correction to what enrahitu published, where
  the platform packages inherited `Apache-2.0` from the meta package by
  copy-paste.

### Versioning

The `@statecrafting/*` packages track their own semver, independent of any
consumer's release tags. Publishing is idempotent: a tag that does not bump
a package must not fail the publish job, so each leg checks npm for the
exact version and skips if present. This is inherited from
enrahitu/018-packaged-chassis and is load-bearing, not incidental.

### Publishing

`.github/workflows/publish.yml` runs a three-leg matrix on version tags:
`darwin-arm64` on `macos-14`, `linux-x64` on `ubuntu-24.04`, `linux-arm64`
on `ubuntu-24.04-arm`. Each leg builds its own binaries natively; there is
no cross-compilation. One designated leg (`darwin-arm64`) additionally
publishes the meta package.

The binaries cannot be built locally for all three platforms from one
machine, so publishing is CI-only by construction. It requires the
`NPM_TOKEN` repository secret and the claimed `@statecrafting` npm scope,
both of which exist as of 2026-07-17.

### The old names

`@enrahitu/toolchain@0.1.0` and its platform packages are live on npm
(published 2026-07-15). They are deprecated with a message pointing at
`@statecrafting/toolchain`, not unpublished: unpublishing breaks anyone who
already resolved them, and npm's 72-hour window makes it unreliable anyway.

Deprecation needs a token with `@enrahitu` scope access. The operator
token in use is granular and scoped to `@statecrafting` alone, so the
deprecation is an operator step recorded in Acceptance, not something this
spec's CI can perform.

## 4. Ownership transfer

Per spec 001 section 4, this move transfers edges rather than deleting
specs. Neither exporting spec retires:

- **enrahitu/008-vendored-encore-toolchain drops `vendor/encore/`** and
  keeps `infra.config.dev.json` and `docker/Dockerfile.base`, which stay in
  enrahitu. The spec itself stays `approved` and `complete`: it remains the
  design record of why Encore is vendored at all (rust core + js runtime via
  napi-rs, no CLI), which is the reason this package exists, and it still
  owns two live files.
- **enrahitu/018-packaged-chassis drops `packages/`** and keeps
  `.github/workflows/publish.yml`, which continues to publish enrahitu's
  own artifacts until spec 003 moves hiqlite-native too.

## 5. Acceptance

1. `@statecrafting/toolchain` and the three platform packages are published
   at `0.1.0` with provenance.
2. The platform packages declare MPL-2.0; the meta package declares
   Apache-2.0.
3. enrahitu builds against `@statecrafting/toolchain@0.1.0` from npm, with
   `vendor/encore/` and `packages/` removed from its tree, and its gates
   pass.
4. statecraft builds against `@statecrafting/toolchain@0.1.0` and its gates
   pass.
5. enrahitu/008 no longer claims `vendor/encore/` and still owns
   `infra.config.dev.json` and `docker/Dockerfile.base`; enrahitu/018 no
   longer claims `packages/`. Neither spec is retired or deleted.
6. **Operator step:** `@enrahitu/toolchain` and its platform packages are
   deprecated on npm with a message naming the new package. Requires an
   `@enrahitu`-scoped token; not performable by this repo's CI.

## 6. Out of scope

- `hiqlite-native`, which moves next (spec 003) and shares this publish
  workflow's matrix.
- Bumping the pinned Encore version. This move preserves `v1.57.9`
  exactly; an upgrade is its own change.
