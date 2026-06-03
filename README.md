# GitHub Actions Docker Private Registry Authentication Issues

This repository demonstrates authentication issues when using GitHub Actions with Docker actions and private registries. The tests showcase when Docker actions fail unexpectedly and when they succeed for different action types.

> **🎯 Key Finding**: GitHub Actions pulls Docker images for actions **before** running authentication steps, causing failures with private registries. This repository provides comprehensive tests and workarounds for this behavior.

## Problem Overview

The problem centers around **when** private images are being pulled. In certain scenarios, GitHub Actions tries to pull images early in the process, but does so **before** authentication has occurred, resulting in a `401 Unauthorized` error.

**Specific Issue**: GitHub Actions automatically pulls Docker images for `using: "docker"` actions during the workflow setup phase, which happens before any workflow steps (including authentication steps) are executed. This timing issue makes it impossible to authenticate before the image pull, causing failures with private registries.

**Impact**: This affects any organization using private container registries (GHCR, ACR, ECR, etc.) and wanting to use the `using: "docker"` action type in their reusable actions.

## What We're Testing

### ✅ **What Works Consistently**

- **Composite actions** using `docker run` commands
- **Local action calls** after `actions/checkout`
- **Direct Docker commands** in workflow steps

### ❌ **What Fails Unexpectedly**  

- **Docker image actions** (`using: "docker"` with `image: "docker://registry/image"`)
- **Remote action calls** without checkout
- **Dockerfile actions** with private base images (inconsistent behavior)

## Test Scenarios

Each test demonstrates the authentication behavior across three Docker action types:

### 1. **Composite Actions** (`using: "composite"`)

- **Expected**: ✅ Always succeeds
- **Reason**: Uses direct `docker run` commands which respect authentication

### 2. **Dockerfile Actions** (`using: "docker"`, `image: "Dockerfile"`)

- **Remote call**: ❌ Fails (triggers a pull before login step)
- **Local call**: ✅ Succeeds (after `actions/checkout` and login)
- **Reason**: In remote calls, a dynamically added step tries to pull the image before authentication is set up, leading to a 401 Unauthorized error. Local calls work because they run after the login step, which has the necessary authentication.

### 3. **Image Actions** (`using: "docker"`, `image: "docker://registry/image"`)

- **Remote call**: ❌ Fails (same issue as Dockerfile actions)
- **Local call**: ✅ Succeeds (only after pre-pulling image)

## Repository Structure

### Test Reusable Actions

```text
test-gh-docker/
├── composite/          # 🧩 Composite action (always works)
├── dockerfile/         # 🐳 Dockerfile action (works locally)
└── image/             # 📦 Image action (works with workarounds)
```

### Workflows

```text
.github/workflows/
├── orchestrator.yml         # 🎯 Calls other workflows (organized)
├── comprehensive.yml        # 📋 All tests in one file (monolithic)
├── composite.yml            # 🧩 Individual: Composite tests
├── dockerfile.yml           # 🐳 Individual: Dockerfile tests  
├── image.yml                # 📦 Individual: Image Reference tests
└── dind.yml                # 🔄 Individual: Docker-in-Docker tests
```

## Example Behavior

After running `docker/login-action@v4` to authenticate with a private registry:

```yaml
- uses: docker/login-action@v4
  with:
    registry: private-registry
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

# ❌ This fails
- uses: my-org/my-docker-action@main  # uses: docker, image: docker://private-registry/image

# ✅ This works - Local call with checkout
- uses: actions/checkout@v6
- uses: ./my-docker-action            # Same action, called locally

# ✅ This works - Composite action with docker run
- uses: my-org/my-composite-action@main  # uses: composite, runs: docker run private-registry/image
```

## Root Cause

GitHub Actions tries to pull/build actions dynamically, which happens before all other steps (including docker login), leading to a failure when the action attempts to pull a private image without authentication.

## Current Workarounds

1. **Use `actions/checkout` before calling Docker actions locally** (to use either `docker` or `image reference` actions)
2. **Pre-pull images with `docker pull`** for image reference actions
3. **Convert to composite actions using `docker run`** (Best solution currently)
4. **Make images public** (not feasible for enterprise — and note it defeats *this* demo, which relies on the image being private)

## Expected Behavior

Docker authentication should work consistently across all action types (`using: "docker"` and `using: "composite"`).

## Impact

This affects enterprise users with private registries and forces:

- Inefficient workarounds
- Inconsistent action behavior
- Inability to use `using: "docker"` actions with private registries
- Additional complexity in CI/CD pipelines

## Environment

- **Runners**: `ubuntu-latest`
- **Registries**: Tested with ACR, GHCR
- **Authentication**: `docker/login-action@v4`
- **GitHub Token**: `GITHUB_TOKEN` with `packages: read` permission

## References

- [josh-ops - Create a Docker Container Action Hosted in a Private Image Registry](https://josh-ops.com/posts/github-actions-docker-actions-private-registry/)
- [moby/moby - Issue 38591](https://github.com/moby/moby/issues/38591) (potentially relevant)

---

## Fork Setup

This repository is designed to be forkable. After forking:

1. **Run `build-images` workflow** — Go to **Actions → Build and Push Images** and trigger it manually (`workflow_dispatch`). This builds the `ubuntu` and `dind` images from `images/` and pushes them to your GHCR namespace:
   - `ghcr.io/YOUR-ORG/YOUR-REPO/ubuntu:latest`
   - `ghcr.io/YOUR-ORG/YOUR-REPO/dind:latest`

2. **Keep the packages PRIVATE** — Newly pushed GHCR packages are private by default, which is exactly what this demo needs. **Do not make them public.** ⚠️ Making them public silently breaks the demo: with a public image there's nothing to authenticate against, so every `(Fails)` job starts *succeeding* and the whole point is lost. (This is what once broke this repo.)

   Verify the image is actually private — an anonymous pull must be denied:

   ```bash
   docker logout ghcr.io
   docker pull ghcr.io/YOUR-ORG/YOUR-REPO/ubuntu:latest   # MUST fail: denied / unauthorized
   ```

3. **Update hardcoded refs** — A few places use a literal `a-magdy/gh-actions-docker-demo` that **cannot** use `${{ ... }}` expressions. After forking, change each to `YOUR-ORG/YOUR-REPO`:
   - `test-gh-docker/dockerfile/Dockerfile` — the `FROM` line
   - `test-gh-docker/image/action.yml` — the `image:` value
   - the remote `uses: a-magdy/gh-actions-docker-demo/test-gh-docker/...@main` refs in `composite.yml`, `dockerfile.yml`, `image.yml`, `dind.yml`, and `comprehensive.yml`

   > Everything else uses `${{ github.repository }}` / `${{ github.repository_owner }}` / `${{ github.actor }}` automatically — no changes needed there.

4. **Run any workflow** — Authentication uses `GITHUB_TOKEN` (built-in, no secrets to configure).

### Expected results

When the demo is working, the overall run is **green** even though several jobs show a red ✗ — the `(Fails)` jobs are marked `continue-on-error: true` so their expected failure doesn't fail the run:

| Job | Result |
|---|---|
| Composite (Succeeds) | ✅ pass |
| Dockerfile (Succeeds) | ✅ pass |
| Image (Succeeds) | ✅ pass |
| DinD - Composite (Succeeds) | ✅ pass |
| Dockerfile (Fails) | ✗ expected — image pulled before login |
| Image (Fails) | ✗ expected — image pulled before login |
| DinD - Image / Dockerfile (Fails) | ✗ expected — same timing problem |

The `(Fails)` jobs should fail at the **image pull during job setup** (a `denied`/`unauthorized` pull), *before* the `docker/login-action` step runs — that's the behavior this repo exists to demonstrate. If a `(Fails)` job passes, the image is probably public.
