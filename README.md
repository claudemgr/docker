# 🤖 claudemgr/docker

Template specifications for CasjaysDev Docker image repositories. Each file is a master template — copied into an image repo as `AI.md` — defining the repo's structure, template system, tooling, runtime conventions, CI/CD, and commit workflow.

## Templates

| File | Repo family | When to use |
|------|-------------|-------------|
| `DOCKERSRC.md` | `dockersrc/{name}` → Docker Hub `casjaysdev/{name}` | OS base images and toolchains (alpine, debian, ubuntu, almalinux, archlinux, web, xorg, go, rust, android) |
| `CASJAYSDEVDOCKER.md` | `casjaysdevdocker/{name}` → Docker Hub `casjaysdevdocker/{name}` | Application images (gitea, opengist, super-productivity, …) built FROM `casjaysdev/*` bases |

## Files

| File | Purpose |
|------|---------|
| `DOCKERSRC.md` | Base image template — source of truth for `dockersrc` repos (PARTs 0–8) |
| `CASJAYSDEVDOCKER.md` | App image template — source of truth for `casjaysdevdocker` repos (PARTs 0–8) |
| `README.md` | This file |
| `LICENSE.md` | Repository license (WTFPL) |

## Highlights

- Org mapping: GitHub `dockersrc` ↔ Docker Hub `casjaysdev` for bases; `casjaysdevdocker` on both systems for apps, which always build FROM `casjaysdev/*` bases
- All generated content comes from `gen-dockerfile`/`gen-script` — the spec defines what is generated vs hand-crafted and which files tooling must never overwrite
- OCI label canon: `image.url` is the browsable `https://hub.docker.com/r/casjaysdev/{name}` page; `image.source`/`documentation` point at the GitHub repo; retired labels stay retired
- The bootstrap/update runbook is not in the spec — it lives in the `dockersrc-bootstrap` agent, keeping the template declarative

## Related

Maintenance procedure: `~/.claude/agents/dockersrc-bootstrap.md` (installed from `claudemgr/config`). Global implementation conventions live in `~/.claude/memory/` (dockerfile_conventions.md, cicd_conventions.md, etc.).

## License

This repository (the templates themselves) is licensed under **WTFPL** — see [LICENSE.md](LICENSE.md).

Images built from repos that follow these templates carry the license declared in each repo's `LICENSE.md`.
