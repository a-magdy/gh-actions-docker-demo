# GitHub Actions — Docker Patterns

> Learnings from debugging Docker actions across multiple registries and runner types.

## Permissions

- Always set `permissions:` at the **workflow level**, not per-job (unless jobs differ)
- Setting **any** explicit `permissions:` block **drops ALL defaults** — always include `contents: read`
- Example of the silent failure: a job with only `packages: read` will lose `contents: read`, breaking `actions/checkout` and remote `uses:` downloads

```yaml
permissions:        # ✅ workflow-level
  contents: read
  packages: read
```

## GHCR Auth

- Login **username**: `${{ github.actor }}` (the runner identity)
- Image **names**: `ghcr.io/${{ github.repository_owner }}/...` (the repo owner)
- `workflow-level env:` **cannot** use `${{ github.repository_owner }}` — inline it in `with:` and `run:` steps
- `workflow_call:` callers must grant `permissions: packages: write` for the called workflow to push images

## DinD (Docker-in-Docker)

Always add `--platform linux/amd64` to `docker run` inside container jobs:

```yaml
- run: docker run --rm --platform linux/amd64 ghcr.io/${{ github.repository_owner }}/myimage
```

**Why**: Docker 28 inside DinD reports the host platform as `linux/amd64/v3`. Multi-arch manifests only
list `linux/amd64` (no `/v3`). Docker can't find an exact match and falls back to `linux/arm64/v8`,
causing `exec format error`.

## Private Image Actions (`image: docker://`)

`image: docker://ghcr.io/org/image` in `action.yml` does **not** authenticate before pulling.
GitHub Actions tries to pull Docker images during workflow setup, **before** any login step runs.

**Workaround — always pre-pull:**
```yaml
- run: docker pull ghcr.io/${{ github.repository_owner }}/ubuntu   # auth already done above
- uses: ./my-docker-action                                          # image: docker://... works now
```

Or use `image: Dockerfile` instead — it builds locally after auth.

## Bootstrap Ordering (first push)

On first push to `main`, all `push: branches: [main]` workflows fire simultaneously. Test workflows
that need GHCR images will fail because images haven't been built yet.

**Fix — orchestrator with `needs:`:**
```yaml
# orchestrator.yml
jobs:
  build-images:
    uses: ./.github/workflows/build-images.yml
    permissions: { contents: read, packages: write }
  test-docker:
    needs: build-images
    uses: ./.github/workflows/docker.yml
    secrets: inherit
```

Also add to each test workflow:
```yaml
on:
  workflow_call:   # called by orchestrator
  push:
    paths-ignore:
      - "images/**"   # skip image-only pushes — orchestrator handles those
```

## Forkability

- Never hardcode registry usernames in workflow `env:` — use `${{ github.repository_owner }}`
- `image: docker://` and Dockerfile `FROM` lines cannot use expressions — add a fork comment:
  ```
  # Forks: change 'org-name' to your org/user after running the build-images workflow.
  ```

## Dockerfile Best Practices

- Pin base images: `alpine:3.20` not `alpine:latest`, `ubuntu:24.04` not `ubuntu:latest`
- Clean apt cache in the same layer: `apt-get install -y pkg && rm -rf /var/lib/apt/lists/*`

## Job Timeouts

Always set `timeout-minutes:` to prevent hung runners:
- Build/push jobs: `15`
- Test jobs: `10`
- DinD jobs: `15`
