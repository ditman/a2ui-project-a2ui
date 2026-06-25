# Publishing Guide for A2UI Web Packages

This guide outlines the workflow for project maintainers publishing web packages
(`@a2ui/*`) to npm through Google's internal Artifact Registry.

## Prerequisites: Authentication

Publishing requires access to Google's internal Artifact Registry. Ensure you
are authenticated with the Google Cloud CLI before starting:

```sh
gcloud auth login
```

The release scripts retrieve this authentication token using
`gcloud auth print-access-token` when needed.

---

## Release Workflow

All scripts should be executed from the repository root. The workflow consists
of a pull request bumping the versions of the packages to release, followed by
publishing from the `main` branch.

### 1. Increment Versions (in a Pull Request)

Before publishing, create a pull request branch and use `increment_version.mjs`
to bump package versions and synchronize locks:

```sh
# Increment patch version automatically (e.g., 0.9.5 -> 0.9.6)
./renderers/scripts/increment_version.mjs web_core

# Set an explicit version
./renderers/scripts/increment_version.mjs lit 0.10.0-beta.1
```

Submit a PR with the updated files (`package.json`, `CHANGELOG.md`, etc...), get
it approved, and merge it into `main`.

**CLI parameters for `increment_version.mjs`:**

- `<package-name>`: The name of the package to update (e.g., `web_core` or `@a2ui/web_core`).
- `[new-version]`: The specific new version to set (e.g., `1.0.1`). If omitted, increments the patch version automatically.
- `--skip-sync`: Skip synchronizing dependent packages (running `yarn install` in dependents).

### 2. Publish to Staging (from `main`)

Once the version bump PR is merged, checkout or pull the latest `main` branch.
Use `publish_npm.mjs` to build, test, and upload the packages to Google's
internal Artifact Registry:

```sh
./renderers/scripts/publish_npm.mjs --package=web_core --package=lit
```

This script publishes requested packages in the correct dependency order (e.g.,
ensuring `web_core` is published before `lit`), runs unit tests, and verifies
that required core packages exist on the registry.

By default, the script runs in dry-run mode to prevent accidental uploads; you
must explicitly pass the `--no-dry-run` flag to actually upload the packages:

```sh
./renderers/scripts/publish_npm.mjs --package=web_core --package=lit --no-dry-run
```

Artifacts are published to: [go/a2ui-oss-exit-gate-artifacts](https://go/a2ui-oss-exit-gate-artifacts).

**CLI parameters for `publish_npm.mjs`:**

- `-p, --package=<name>`: Package(s) to publish. Can be specified multiple times. Accepts short names (e.g., `web_core`) or scoped names (e.g., `@a2ui/web_core`).
- `--no-dry-run`: Actually publish the packages and obtain fresh auth tokens. By default, the script runs in dry-run mode.
- `--skip-tests`: Skip building and testing packages before publishing. Not recommended.
- `--check-core-dependencies` / `--no-check-core-dependencies`: Verify or skip verifying that core dependencies are also being published.

### 3. Trigger Public Release (from `main`)

Once packages are verified in staging on `main`, upload a release manifest to
trigger the internal Exit Gate pipeline, which publishes them to npm:

```sh
./renderers/scripts/upload_manifest.mjs --package=web_core --package=lit
```

Similarly to `publish_npm.mjs`, this script runs in dry-run by default, so to
actually trigger the release you must pass the `--no-dry-run` flag:

```sh
./renderers/scripts/upload_manifest.mjs --package=web_core --package=lit --no-dry-run
```

You will receive confirmation emails from Exit Gate and npm reporting on the
progress of the actual publishing, which normally takes around 5 minutes.

**CLI parameters for `upload_manifest.mjs`:**

- `-p, --package=<name>`: Package(s) to trigger release for (e.g., `--package=lit`). Can be specified multiple times.
- `--no-dry-run`: Actually trigger the public release via Exit Gate. By default, the script runs in dry-run mode.

---

## What is the `publish:package` yarn script doing?

A2UI web packages depend on each other via `workspace:*` links during development. When `publish_npm.mjs` invokes a package's `publish:package` target, the following preparation steps occur:

1. **Build & Metadata Transformation**: `prepare-publish.mjs` copies build output into `dist/`, replaces internal `workspace:` protocols with absolute semantic version ranges (e.g., `^0.10.3`), and strips development scripts/dependencies.
2. **Boundary Isolation**: Because the root workspace config excludes `dist/` (`!**/dist`), an empty `yarn.lock` is initialized inside `dist/` to establish it as an independent package boundary.
3. **Clean Upload**: `yarn npm publish --access public` executes strictly inside `dist/`, ensuring only clean production assets are uploaded.

---

## Troubleshooting

- **Dirty Working Tree Warnings**: If build artifacts or temporary files
  persist, run `yarn clean:all` from the monorepo root and try again.
