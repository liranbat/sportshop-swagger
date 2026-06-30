# sportshop-swagger

OpenAPI 3.0 contracts for the Sportshop backend — one YAML per domain. The
Sportshop services generate code from these specs at build time; this repo
is the single source of truth for the API surface.

## Layout

Every `*.yaml` (or `*.yml`) at the repo root is treated as a deliverable
and gets packaged into the release zip. Today's specs are all named with
an `-api.yaml` suffix (one OpenAPI 3.0 document per domain), but the
workflow doesn't require that suffix — it'll pick up anything with a
YAML extension at the root.

Adding a new spec = dropping a new `*.yaml` file at the root. The
release workflow picks it up automatically on the next publish.
(Subdirectories aren't scanned — workflow files under `.github/` are
intentionally excluded.)

## Releases

Every push to `main` and every successful manual `workflow_dispatch` on
`main` produces a versioned release on the GitHub Releases page. The
release contains a single asset: `sportshop-swagger-<version>.zip`, with
all root-level `*.yaml` / `*.yml` files at the zip root.

**Version scheme:** `{branch}.1.0.<github.run_number>`. The patch number
bumps monotonically per workflow run, prefixed by the source branch — so
consecutive publishes from `main` produce `main.1.0.5`, `main.1.0.6`,
`main.1.0.7`, etc. Manually re-dispatching on the same commit produces a
fresh version (e.g. `main.1.0.8` on the same SHA as `main.1.0.7`). The
branch prefix means versions stay collision-free if other branches are
ever opted into publishing.

**Tag:** each release tags the commit it was built from with the version
verbatim — no `v` prefix (e.g. `main.1.0.5`). Tags are created by
`gh release create` as part of the publish step, so a release and its tag
are always in sync.

**Anonymous downloads:** the repo is public, so the release zip is
fetchable without any auth:

```
https://github.com/liranbat/sportshop-swagger/releases/download/<version>/sportshop-swagger-<version>.zip
```

## CI pipeline

`.github/workflows/release.yml` has a single `release` job that runs on
every push to `main` and every successful manual `workflow_dispatch` on
`main`. It computes the version, zips every root-level YAML, and creates
the GitHub Release — which also creates the `<version>` git tag in the
same atomic step.

Workflow validity is enforced downstream: any malformed YAML will fail
the backend's `openapi-generator-maven-plugin` build, which the backend's
own CI catches before merge.

## Consumption from `sportshop-backend`

The backend pins a version in its `pom.xml`:

```xml
<properties>
  <sportshop-swagger.version>main.1.0.5</sportshop-swagger.version>
</properties>
```

A `maven-antrun-plugin` execution in the `initialize` phase downloads
`sportshop-swagger-${sportshop-swagger.version}.zip` from the release URL
above and unzips into `target/openapi/`. The existing
`openapi-generator-maven-plugin` executions then run against the YAMLs
in `target/openapi/` exactly as before — only the source of those files
changes.

To pull a new version into the backend: bump the property, rebuild.

## Local iteration

Default builds always download from the release zip — there's no
auto-detected sibling-folder fallback. For local YAML iteration, the
backend `pom.xml` ships a `local-specs` Maven profile that swaps the
download for a copy from `../sportshop-swagger/`.

**Fast loop** (no publish, no version bump):

1. Edit YAMLs in your local `sportshop-swagger/` clone.
2. From `sportshop-backend/`, build with the profile activated:

   ```
   mvn install -Plocal-specs
   ```

   The build copies `../sportshop-swagger/*.yaml` straight into
   `target/openapi/` and skips the network entirely. Re-run as often as
   you like.

**Promotion loop** (when ready to share):

1. Push the YAML changes to a publishing branch.
2. Wait for the workflow to publish a new version (e.g. `main.1.0.8`).
3. Bump `<sportshop-swagger.version>` in `sportshop-backend/pom.xml` to
   that version.
4. Drop the `-Plocal-specs` flag on your next build — the default
   release-zip path picks up the new version.
