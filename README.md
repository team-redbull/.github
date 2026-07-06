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
