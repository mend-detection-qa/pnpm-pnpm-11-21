# pnpm-1121-lockfile-peers-git

Probe generated: 2026-08-09
PM version under test: pnpm 11.21.0
Pattern: `pnpm-1121-lockfile-peers-git`
Categories: `lockfile_format`, `tree_structure`, `registry_source`

## Feature exercised

Three pnpm 11.21 behaviors exercised in a single workspace probe:

1. **lockfile_format — Catalog entries and lockfile pruning.**
   `pnpm-workspace.yaml` declares a `catalog:` block with `react`,
   `react-dom`, and `zod`. Workspace packages reference these via
   `"react": "catalog:"` in their `package.json`. The lockfile
   records a top-level `catalogs:` section with the resolved
   versions; the `importers` section records `specifier: 'catalog:'`
   against each entry. pnpm 11.21 pruning: only packages reachable
   from at least one importer survive in `packages` and `snapshots`.
   In this probe, `zod` is only used by the root importer — if the
   root's devDependency were removed, `zod` would be pruned from
   both sections immediately on the next install.

2. **tree_structure — Peer dependency fast-path.**
   `@probe/ui-button` declares `react@^18.3.0` as a peer dependency.
   `@probe/app-web` depends on `@probe/ui-button` (via `workspace:*`)
   AND also directly depends on `react@18.3.1` (from catalog).
   Because `autoInstallPeers: true` and app-web's `react@18.3.1`
   satisfies ui-button's peer range exactly, pnpm 11.21 takes the
   fast-path: no virtual snapshot fork is created. The snapshot key
   in the lockfile is plain `react@18.3.1`, NOT
   `react@18.3.1(peer-dep-foo@version)`. This is the critical
   difference from older pnpm behavior where even trivially-satisfied
   peers generated virtual package entries.

3. **registry_source — Git URL normalization.**
   `@probe/app-web` depends on `is-positive` via
   `"github:sindresorhus/is-positive#3.1.0"` shorthand. pnpm 11.21
   records this in the lockfile as a tarball resolution pointing to
   `https://codeload.github.com/...` with the commit SHA, rather
   than as a `git+ssh://` or `git+https://` URL. This exercises
   Mend's handling of tarball-backed git deps: the `resolution` block
   has both `tarball` and `commit` fields, not just `integrity`.

## Workspace layout

```
pnpm-1121-lockfile-peers-git-20260809-143205/
├── package.json                 # root, devDep: zod (catalog:)
├── pnpm-workspace.yaml          # catalog: react, react-dom, zod
├── pnpm-lock.yaml               # v9.0 with catalogs: section
├── .npmrc
├── .whitesource
├── README.md
├── expected-tree.json
└── packages/
    ├── ui-button/
    │   └── package.json         # peerDeps: react; devDeps: react (catalog:)
    └── app-web/
        └── package.json         # deps: @probe/ui-button (workspace:*),
                                 #       is-positive (github:), react, react-dom
```

## Expected dependency tree summary

- Root (`pnpm-1121-lockfile-peers-git-probe`): direct devDep `zod`
- `packages/ui-button` (`@probe/ui-button`): no runtime deps, only
  devDep on `react@18.3.1`
- `packages/app-web` (`@probe/app-web`): direct deps
  `@probe/ui-button` (local), `is-positive` (git/tarball),
  `react@18.3.1`, `react-dom@18.3.1`
- `react-dom@18.3.1` transitively depends on `react`, `loose-envify`,
  `scheduler`
- `react@18.3.1` transitively depends on `loose-envify`
- `loose-envify@1.4.0` depends on `js-tokens`
- `scheduler@0.23.2` depends on `loose-envify`
- `zod@3.22.4`: no dependencies
- `is-positive@3.1.0`: no dependencies (tarball-backed git source)

## What Mend must detect

- `zod@3.22.4` resolved correctly from `catalog:` specifier (not the
  string literal `"catalog:"`).
- `react@18.3.1` and `react-dom@18.3.1` resolved from `catalog:` in
  each workspace package that references them.
- `@probe/ui-button` reported as `source: "local"` (workspace link).
- `is-positive@3.1.0` reported as `source: "git"` with commit SHA
  `c87a5e76c37deca5a5b4b8a79c70d67c11b07ac5`.
- No duplicate/virtual peer forks: `react@18.3.1` appears exactly
  once in the packages map (fast-path took effect).

## Mend failure modes to watch

- Catalog entries: `catalog:` reported as the version string.
- Catalog pruning test: if the root `zod` devDep is removed and the
  probe regenerated, Mend should no longer report `zod` (pruned from
  lockfile).
- Peer fast-path: older parsers expecting virtual snapshot keys like
  `react@18.3.1(@probe/ui-button@1.0.0)` may fail to find `react`
  under its plain key.
- Git source: Mend may report `is-positive` as `source: "registry"`
  if it mistakes the tarball URL for an npm registry tarball. The
  distinguishing signal is the `commit` field alongside `tarball` in
  the resolution block.
- SSH normalization: if someone specifies
  `git+ssh://git@github.com/sindresorhus/is-positive.git`, pnpm 11.21
  normalizes it to the same codeload tarball URL; Mend should produce
  the same `git` source entry regardless of which input form was used.

## Mend config

Bucket A — default-emit with `pnpm: "11.21.0"` and
`node: "20.18.0"` pinned in `.whitesource`. pnpm has no dynamic
version detection from the manifest; the `.whitesource` pin
ensures `install-tool` provisions exactly pnpm 11.21.0 for this
probe so the lockfile format and peer fast-path behavior match
what the tree was generated against.

No `whitesource.config` is present; `configMode` is `"AUTO"`.

## Resolver knowledge notes

The Mend UA resolver for pnpm is `PnpmLockCollector` with
`PnpmParserV9Impl`. Key facts from the live resolver file
(fetched 2026-08-09, SHA 35e40487698f402b1015f97902d7decebe1cc97d):

- Detects from `lockfileVersion` field; routes to V9 parser for
  `lockfileVersion: '9.0'`.
- V9 parser reads direct deps from `importers`, resolution
  metadata from `packages`, and transitive dep lists from
  `snapshots`.
- Catalog support: the resolver must read `catalogs:` block in
  the lockfile and resolve `specifier: 'catalog:'` entries to
  actual versions before building the tree.
- Git resolution: `{tarball: ..., commit: ...}` in `packages`
  maps to `source: "git"` in the expected tree.
- SHA1 enrichment: pnpm SHA1 is obtained via `pnpm ls --json`.

The new `catalogs:` top-level section in pnpm 11.x lockfiles
is NOT mentioned in the resolver file's V9 parser description
(as of the fetch date). This means catalog specifier resolution
is exploratory — the probe README documents this gap so the
downstream comparator treats catalog-specifier failures as
unconfirmed bugs rather than regression breaks.
