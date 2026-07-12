# .github

Org-wide shared GitHub Actions reusable workflows and community defaults.

## `ghcr-build-push.yml`

Shared build/version/push flow for team-redbull service repos: bumps a
semver git tag, builds a Dockerfile, pushes it to GHCR, then bumps an
`{repository, tag}` block in the repo's Helm chart values and commits that
change back.

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
      dockerfile: Dockerfile                # optional; this is the default (repo-root Dockerfile, repo-root build context)
      helm-image-path: image                # optional; this is the default (yq path to the {repository, tag} block)
    secrets: inherit
```

Change the shared flow once here — every caller picks it up on its next
run (pin `@main` to a tag, e.g. `@v1`, if you want controlled rollout
instead of always-latest).

### Multiple images from one repo

A repo that publishes more than one image (e.g. a Temporal "brain" +
"limb" pair, each its own Dockerfile and its own `{repository, tag}` block
in the same `values.yaml`) calls the reusable workflow once per image, with
`dockerfile` and `helm-image-path` set per call. **Chain them with `needs:`
so they run sequentially, not in parallel** — every call bumps the same
shared `vX.Y.Z` git tag sequence and commits+pushes to the same branch;
parallel calls would race on both the tag and the push.

```yaml
jobs:
  build-brain:
    uses: team-redbull/.github/.github/workflows/ghcr-build-push.yml@main
    with:
      image-name: connectivity-workflow
      dockerfile: workflows/Dockerfile
      helm-values-path: helm/connectivity/values.yaml
      helm-image-path: workflowWorker.image
    secrets: inherit

  build-activity:
    needs: build-brain   # sequential: avoids racing build-brain on the git tag/push
    uses: team-redbull/.github/.github/workflows/ghcr-build-push.yml@main
    with:
      image-name: connectivity-activity
      dockerfile: activities/connectivity/Dockerfile
      helm-values-path: helm/connectivity/values.yaml
      helm-image-path: activityWorker.image
    secrets: inherit
```

## GHCR image cleanup

Two workflows keep GHCR from accumulating unbounded per-commit images from
non-main branches. Both require a `GHCR_CLEANUP_TOKEN` secret (a classic
PAT with `read:packages` + `delete:packages`, added as an org secret).
**This secret must be scoped to `team-redbull/.github` itself** (not just
the repos that get cleaned up) — the scheduled sweep runs in *this* repo's
own Actions context, so without it here the "fully centralized" part below
silently has nothing to delete with. Scope it to any other repo too if
that repo also wants the on-delete piece.

Main's `vX.Y.Z` release tags are never touched by either workflow — both
explicitly exclude anything matching `^v[0-9]+\.[0-9]+\.[0-9]+$` before
considering what to prune.

- **`ghcr-cleanup-scheduled.yml`** — runs nightly (+ `workflow_dispatch`,
  which also accepts a `keep` input if you want to keep more than the
  default 1 tag per branch) against *every* container package in the org,
  keeping only the newest tag(s) per branch slug. This one is fully
  centralized: it discovers packages via the org Packages API on its own,
  so **no repo needs to reference it or change anything** to get this
  behavior.

- **`ghcr-cleanup-on-delete.yml`** — purges all tags for a branch the
  moment it's deleted (not on PR close — only when the branch ref itself
  is actually deleted). This one *does* need a small addition per repo,
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

  **If your repo isn't on `ghcr-build-push.yml` yet** (still has its own
  bespoke build job, publishing elsewhere), don't swap that job to adopt
  cleanup — just add the `delete` trigger and `cleanup-on-delete` job
  alongside your existing `build` job, guarded the same way
  (`if: github.event_name == 'push'` on your existing job so it doesn't
  fire on delete events too). `ghcr-cleanup-on-delete.yml` only manages
  GHCR tags for the `image-name` you give it; it doesn't care what your
  build job does or what registry it publishes to.
