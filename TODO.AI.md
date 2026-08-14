# TODO.AI.md

Live-repo issues found during the COMPOSEMGR.md audit (2026-08-07).
All are fix-on-touch / fix-at-sweep items in `composemgr/*` repos — not
bugs in this repo's templates.

- [ ] `composemgr/rustfs`: `container_name: rustfs-server` mismatches the
  service key `app` and its own `CONTAINER_NAME: rustfs-app` env — rename
  to `rustfs-app` per COMPOSEMGR.md PART 3 canon (fix on touch)
- [ ] `composemgr/icecast`: `x-env-file` uses the bare list form
  (`- .env`), which breaks the zero-config guarantee when the files are
  absent — convert to `- path: .env` / `required: false` form per
  COMPOSEMGR.md PART 3 (fix on touch)
- [ ] 286 of 300 `composemgr/*` repos carry legacy traefik labels with
  invalid `Host($(${BASE_HOST_NAME:-$HOSTNAME}))` matcher syntax and
  labels enabled by default — correct to backtick `Host(` form and
  labels-off default per COMPOSEMGR.md PARTs 3–4 (fix at AI.md sweep,
  on the user's word)
