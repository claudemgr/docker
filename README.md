# 🤖 claudemgr/docker

Template specifications for CasjaysDev Docker image repositories. Each file is a master template — copied into an image repo as `AI.md` — defining the repo's structure, template system, tooling, runtime conventions, CI/CD, and commit workflow.

## Templates

| File | Repo family | When to use |
|------|-------------|-------------|
| `DOCKERSRC.md` | `dockersrc/{name}` → Docker Hub `casjaysdev/{name}` | OS base images and toolchains (alpine, debian, ubuntu, almalinux, archlinux, web, xorg, go, rust, android) |
| `CASJAYSDEVDOCKER.md` | `casjaysdevdocker/{name}` → Docker Hub `casjaysdevdocker/{name}` | Application images (gitea, opengist, super-productivity, …) built FROM `casjaysdev/*` bases |
| `COMPOSEMGR.md` | `composemgr/{name}` | docker compose stacks (vaultwarden, gitea, arrstack, …) — hand-maintained, no generator tooling, no CI/CD |

## Files

| File | Purpose |
|------|---------|
| `DOCKERSRC.md` | Base image template — source of truth for `dockersrc` repos (PARTs 0–9) |
| `CASJAYSDEVDOCKER.md` | App image template — source of truth for `casjaysdevdocker` repos (PARTs 0–9) |
| `COMPOSEMGR.md` | Compose stack template — source of truth for `composemgr` repos (PARTs 0–8) |
| `README.md` | This file |
| `LICENSE.md` | Repository license (WTFPL) |

## Highlights

- Org mapping: GitHub `dockersrc` ↔ Docker Hub `casjaysdev` for bases; `casjaysdevdocker` on both systems for apps, which always build FROM `casjaysdev/*` bases; `composemgr` repos deploy images, never build them
- Compose stacks: labels are optional and off by default — cloudflare/traefik label blocks are added only on request from the templates in COMPOSEMGR.md PART 4; networks default to one stack-private network
- Every master ends with an examples PART excerpted from real repos (go toolchain 05-custom.sh, gitea init.d scripts, vaultwarden/gitea compose stacks)
- All generated content comes from `gen-dockerfile`/`gen-script` — the spec defines what is generated vs hand-crafted and which files tooling must never overwrite
- OCI label canon: `image.url` is the browsable `https://hub.docker.com/r/casjaysdev/{name}` page; `image.source`/`documentation` point at the GitHub repo; retired labels stay retired
- The bootstrap/update runbook is not in the spec — it lives in the `dockersrc-bootstrap` agent, keeping the template declarative

## Related

Maintenance procedure: `~/.claude/agents/dockersrc-bootstrap.md` (installed from `claudemgr/config`). Global implementation conventions live in `~/.claude/memory/` (dockerfile_conventions.md, cicd_conventions.md, etc.).

## License

This repository (the templates themselves) is licensed under **WTFPL** — see [LICENSE.md](LICENSE.md).

Images built from repos that follow these templates carry the license declared in each repo's `LICENSE.md`.
