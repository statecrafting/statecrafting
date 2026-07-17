# statecrafting

The shared native packages behind the statecraft product family: the
napi-rs addons and the Encore build toolchain that statecraft, enrahitu,
and chancery all consume.

## Status

Born governed, pre-code. The spec spine landed before the packages
(`specs/000-bootstrap/`); `specs/001-packages-thesis/` records what is
migrating here, why, and in what order. The packages themselves arrive one
per spec.

## Why this repo

These packages were scattered across three repositories under three npm
scopes, and the scope had stopped describing ownership: `@enrahitu/toolchain`
is a build dependency of statecraft, which is not enrahitu, and
`@chancery/kernel-native` had no governing spec at all. One repo, one
`@statecrafting/*` scope, one spine.

| Package | Comes from | License |
|---|---|---|
| `@statecrafting/toolchain` | enrahitu `packages/toolchain/` | Apache-2.0 (vendored Encore stays MPL-2.0) |
| `@statecrafting/hiqlite-native` | enrahitu `addon/` | Apache-2.0 |
| `@statecrafting/kernel-native` | chancery `kernel-addon/` | Apache-2.0 |
| `@statecrafting/governance-native` | statecraft `addon/governance-native/` | AGPL-3.0 |
| `@statecrafting/fleet-native` | statecraft `addon/fleet-native/` | AGPL-3.0 |

## Licensing

**Read `specs/001-packages-thesis/` before adding or relicensing a package.**
The root is Apache-2.0 and that is the default for new packages, but the
root license is a default, not a claim: three licenses coexist here on
purpose. The permissive packages are the ones stamped customer apps
consume, and they must stay unencumbered. `governance-native` and
`fleet-native` are control-plane internals that keep statecraft's AGPL
shield, and no permissive package may depend on them.

Every package declares its own license in its manifest and carries its own
LICENSE file. Never infer a package's license from the root or from a
neighbour.

## Governance

Governed by [spec-spine](https://github.com/statecrafting/spec-spine)
(`cargo install spec-spine-cli`):

```bash
spec-spine compile   # specs -> .derived/spec-registry/by-spec/
spec-spine index     # code linkage -> .derived/codebase-index/
spec-spine lint      # corpus conformance
spec-spine couple --base origin/main --head HEAD   # the PR coupling gate
```

Read `.derived/**` only through `spec-spine` subcommands; the shards are
compiler-owned.
