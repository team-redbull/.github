# .github

Org-wide shared GitHub Actions reusable workflows and community defaults.

## `ghcr-build-push.yml`

Shared build/version/push flow for team-redbull service repos: bumps a
semver git tag, builds the Dockerfile in the repo root, pushes it to GHCR,
then bumps `image.repository`/`image.tag` in the repo's Helm chart values
and commits that change back.

Each repo needing this should have a thin caller workflow, e.g.
`.github/workflows/build.yml`:

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    uses: team-redbull/.github/.github/workflows/ghcr-build-push.yml@main
    with:
      image-name: segments-manager          # optional; defaults to the repo name
      helm-values-path: deploy/helm/values.yaml   # optional; this is the default
    secrets: inherit
```

Change the shared flow once here — every caller picks it up on its next
run (pin `@main` to a tag, e.g. `@v1`, if you want controlled rollout
instead of always-latest).

## GHCR image cleanup

Two workflows keep GHCR from accumulating unbounded per-commit images from
non-main branches. Both require a `GHCR_CLEANUP_TOKEN` secret (a classic
PAT with `read:packages` + `delete:packages`, added as an org secret and
scoped to whichever repos need it).

- **`ghcr-cleanup-scheduled.yml`** — runs nightly (+ `workflow_dispatch`)
  against *every* container package in the org, keeping only the newest
  tag per branch slug. This one is fully centralized: it discovers
  packages via the org Packages API on its own, so **no repo needs to
  reference it or change anything** to get this behavior.

- **`ghcr-cleanup-on-delete.yml`** — purges all tags for a branch the
  moment it's deleted. This one *does* need a small addition per repo,
  because GitHub only fires the `delete` event for workflows registered in
  the repo where the branch lived — a reusable `workflow_call` workflow is
  never invoked on its own. Add this to your repo's existing
  `build.yml` (not a new file, just extra triggers/jobs in the one you
  already have):

  ```yaml
  name: Build and Push

  on:
    push:
      branches: [main]
    delete:

  jobs:
    build:
      if: github.event_name == 'push'
      uses: team-redbull/.github/.github/workflows/ghcr-build-push.yml@main
      with:
        image-name: segments-manager
      secrets: inherit

    cleanup-on-delete:
      if: github.event_name == 'delete'
      uses: team-redbull/.github/.github/workflows/ghcr-cleanup-on-delete.yml@main
      with:
        image-name: segments-manager
      secrets: inherit
  ```
