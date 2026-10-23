# Lab 10 — Cloud: Registry Release CI + Public Deploy

- Student: Arina Nikolaeva (Nik-ari-ai)
- Fork: https://github.com/Nik-ari-ai/DevOps-Intro
- Branch: feature/lab10

## Task 1 — Push the image to a real registry via CI

Release workflow builds the QuickNotes image and pushes it to GHCR on every `v*` tag.

### `.github/workflows/release.yml`

```yaml
name: release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Log in to GHCR
        uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@dc802804100637a589fabce1cb79ff13a1411302 # v6.2.0
        with:
          images: ghcr.io/nik-ari-ai/quicknotes
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
            type=raw,value=latest
      - name: Build and push
        uses: docker/build-push-action@53b7df96c91f9c12dcc8a07bcb9ccacbed38856a # v7.3.0
        with:
          context: ./app
          push: true
          platforms: linux/amd64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

- All actions are **SHA-pinned** (comment holds the human tag). SHAs resolved via `gh api repos/<action>/commits/<latest-tag>`.
- `permissions: packages: write` + login with the job's `GITHUB_TOKEN` — no PAT needed to push to GHCR.
- `metadata-action` derives the tag set from the git tag; `build-push-action` builds `./app` (multi-stage distroless from Lab 6, added to this branch) and pushes.

### Evidence

Tag `v1.0.0` triggered the workflow; final run **succeeded**:

```
$ gh run view -R Nik-ari-ai/DevOps-Intro <id> --json conclusion --jq .conclusion
success

JOBS
✓ release
  ✓ Checkout
  ✓ Log in to GHCR
  ✓ Docker metadata
  ✓ Build and push
```

Image published at `ghcr.io/nik-ari-ai/quicknotes` with tags **`1.0.0`, `1.0`, `latest`, `sha-<commit>`**; package set **Public**. Screenshot: `submissions/lab10-ghcr-package.png`.

> Engineering trail (kept honest): the first two tag pushes failed and were fixed before the green run — (1) `cache-to: type=gha` is unsupported on the default buildx `docker` driver → removed the GHA cache; (2) `Dockerfile not found` because `app/Dockerfile` was not on this branch (each `feature/labN` is cut from `main`) → added the Lab 6 multi-stage Dockerfile to `app/`. Third run green.

## Task 2 — Public deploy

> **Deviation from the brief (documented).** The brief targets Hugging Face Spaces. HF now gates free CPU Docker/Gradio Spaces behind a PRO subscription — `create_repo(..., space_sdk='docker')` returns `402 Payment Required` ("hosting Gradio and Docker Spaces on free cpu-basic requires a PRO subscription"). Static Spaces are free but cannot run a Go HTTP server. So the same GHCR image was deployed to **Render** (free Web Service, no card) instead — this keeps the intended registry → public-URL pipeline and gives a real public URL.

Render **Web Service → Deploy an existing image**: `ghcr.io/nik-ari-ai/quicknotes:latest`, Free instance. Render auto-detected the container port (`New primary port detected: 8080`) and routed to it.

Public URL: **https://quicknotes-latest.onrender.com**

### Evidence (live)

```
$ curl -s https://quicknotes-latest.onrender.com/health
{"notes":4,"status":"ok"}

$ curl -s https://quicknotes-latest.onrender.com/notes | jq 'length'
4

$ curl -s -X POST -d '{"title":"render","body":"deployed via GHCR image"}' \
    https://quicknotes-latest.onrender.com/notes
{"id":5,"title":"render","body":"deployed via GHCR image","created_at":"2026-09-20T21:17:01.403080007Z"}

$ curl -s https://quicknotes-latest.onrender.com/metrics | head -3
# HELP quicknotes_notes_total Notes currently stored.
# TYPE quicknotes_notes_total gauge
quicknotes_notes_total 5
```

Render deploy log: `Deploy succeeded | Live`, `quicknotes listening on :8080 (notes loaded: 4)`, `Your service is live`. Screenshots: `submissions/lab10-render-live.png`, `submissions/lab10-notes-json.png`.

## Bonus — Cloudflare Tunnel + latency comparison

A second copy of the **same GHCR image** runs locally and is exposed through a Cloudflare **quick tunnel** (no account, ephemeral `trycloudflare.com` URL):

```
$ docker run -d --rm -p 8090:8080 --name qn-local ghcr.io/nik-ari-ai/quicknotes:latest
$ cloudflared tunnel --url http://localhost:8090
TUNNEL=https://states-bracelets-wing-jury.trycloudflare.com
$ curl -s https://states-bracelets-wing-jury.trycloudflare.com/health
{"notes":4,"status":"ok"}
```

### Measured latency (`curl -w '%{time_total}'` on `/health`)

| Target | Backend | Warm avg (10 req) | Range |
|---|---|---|---|
| Cloudflare tunnel | local container (Mac) | **0.318 s** | 0.276–0.481 |
| Render | managed (free) | **0.421 s** | 0.358–0.541 |

**Cold-start (scale-to-zero):** attempted twice, leaving Render idle ~15 min and ~30 min with no browser traffic. Both times the first request came back warm — `0.645 s` and `0.435 s` (`ttfb` ≈ `total`, i.e. no container boot). So **a real spin-up was not reproduced** in these windows; the numbers above are honest and un-fabricated. Render's documented behaviour for free instances is spin-down after ~15 min inactivity with a cold start of tens of seconds; in practice the instance stayed warm longer than documented during the test.

### Reading the numbers

- **Warm**, the tunnel (0.318 s) was slightly faster than Render (0.421 s): the tunnel's backend is a local process on the same machine, so the only network cost is the round-trip to the nearest Cloudflare edge; Render adds managed-platform hops.
- **Trade-off.** The tunnel is always warm but tied to a running dev machine and a throwaway URL — good for a quick demo, not a deployment. Render gives a stable managed URL and TLS for free, at the cost of free-tier scale-to-zero (a cold first request after real idle). Scale-to-zero trades first-request latency for $0 idle cost — right for a hobby service, wrong for anything latency-sensitive, where you keep at least one instance warm.
