# Repo context (read first)

This repo is a **standalone HAOS custom add-on** that runs N self-hosted
GitHub Actions runners (one per configured entry, each pointing at its
own GitHub repo). Multi-tenant since v2.0.0.

- **GitHub:** https://github.com/duncanjaffrey77-collab/ha-staff-management-runner
- **Original use case:** the `duncanjaffrey77-collab/staff-management` repo
  exhausted its hosted Actions compute budget; this add-on relieved it.
- **Current scope:** any number of repos via the `runners:` list option.

## Layout

```
.
├── README.md                  ← user-facing install + ops guide
├── CLAUDE.md                  ← this file
├── repository.yaml            ← HAOS add-on repo manifest (top-level)
└── github-runner/             ← the add-on package
    ├── Dockerfile             ← Debian Bookworm + Node 20 + wrangler + runner template
    ├── build.yaml             ← base image pin
    ├── README.md              ← in-UI documentation tab
    ├── config.yaml            ← metadata + multi-runner options schema
    ├── translations/en.yaml   ← Options form UI labels
    └── rootfs/etc/services.d/runner/run   ← multi-runner supervisor (must be 0755)
```

## Common workflows

### Release a new version

1. Make the change.
2. Bump `version:` in [github-runner/config.yaml](github-runner/config.yaml)
   (semver — patch for bugs, minor for features, **major for breaking
   schema changes** like v1.x → v2.0).
3. Commit + push to `main`.
4. Tag: `git tag vX.Y.Z && git push --tags`.
5. In HA: Add-on Store → ⋮ → Reload. Tile gains **Update**.

**Most common shipping mistake:** forgetting to bump `version:`. HA only
re-pulls when the version changes; without a bump, users never see the
update.

### Bump the runner binary version

GitHub releases new runners every ~2 weeks
(https://github.com/actions/runner/releases).

1. Edit [Dockerfile](github-runner/Dockerfile) — bump
   `ARG GITHUB_RUNNER_VERSION="..."`.
2. Bump `version:` in `config.yaml`. Image rebuild propagates the new
   binary as the new `/opt/runner/` template; runners pick it up on
   next start via the per-instance `cp -r` (existing
   `/opt/runners/<name>/` dirs are wiped by image rebuild and recreated
   from the new template).
3. Commit + push + tag.

### Add support for a new option field

1. Update `schema:` and `options:` defaults in `config.yaml`.
2. Update the corresponding label/help text in `translations/en.yaml`.
3. Update the launch script if behaviour depends on it.
4. Bump `version:` (minor unless it changes behaviour of existing
   options).

## Things to **not** do

- **Don't commit registration tokens, PATs, or `.credentials/` files.**
- **Don't rename `github-runner/`** — HA Supervisor uses the directory
  name as the add-on slug-context.
- **Don't edit `repository.yaml` `url:`** — must match the repo's
  origin URL.
- **Don't add CI workflows in this repo that target `ubuntu-latest`.**
  This repo's purpose is to eliminate hosted runner usage. Either
  target this add-on's runners (after they're configured for this repo
  itself, which is a chicken-and-egg situation we haven't done), or
  don't add CI at all.
- **Don't reintroduce a deregister-on-stop trap.** It wipes the runner
  server-side on every Stop, forcing fresh tokens on every Start.
- **Don't write per-runner state outside `/data`.** HA Supervisor
  recreates the add-on container on Stop/Start; only `/data` persists.
  This is why credentials are mirrored to `/data/runner-credentials/<name>/`.

## File mode caveat (Windows)

The s6 launch script [run](github-runner/rootfs/etc/services.d/runner/run)
**must be mode 0755** in the published artifact. Verify after any edit:

```bash
git ls-files --stage github-runner/rootfs/etc/services.d/runner/run
# expect:  100755 <sha> 0  github-runner/rootfs/etc/services.d/runner/run
```

If it shows `100644`:
```bash
git update-index --chmod=+x github-runner/rootfs/etc/services.d/runner/run
git commit -m "fix: mark run script executable"
```

The `.gitattributes` in the repo root enforces LF endings on the script
and other add-on files, so Windows editors don't break shell scripts at
the next checkout.

## Runner state model (important context)

Each configured runner has three persistent paths:

| Path | Lifetime | Purpose |
|---|---|---|
| `/opt/runner/` | image rebuild | Template install — read-only at runtime. Built into the Docker image. |
| `/opt/runners/<name>/` | container layer | Per-instance install — `cp -r` from `/opt/runner/` on first start of this entry. Wiped by image rebuild. |
| `/data/runner-credentials/<name>/` | host volume | Persisted `.credentials`, `.credentials_rsaparams`, `.runner` — survives container recreation AND image rebuild. |
| `/data/_work/<name>/` | host volume | Workflow job working dir. Persists; can grow large. |

**Boot sequence per runner:**
1. If `/opt/runners/<name>/` is missing → `cp -r /opt/runner /opt/runners/<name>/`.
2. If `/opt/runners/<name>/.credentials` missing AND
   `/data/runner-credentials/<name>/.credentials` exists → restore from
   `/data`.
3. If still no credentials → register using the entry's
   `registration_token`. Mirror credentials to `/data/runner-credentials/<name>/`.
4. `exec ./run.sh` in the per-runner install dir.

The supervisor `wait`s for all runners; if any exit, the others keep
working. If all exit, the supervisor exits and s6 restarts the whole
service.

## Historical: v1.x → v2.x → v3.0 migration

- v1.x was single-runner with flat options (`github_repo`,
  `github_runner_token`, `runner_name`, `runner_labels`) and stored
  credentials at `/data/runner-credentials/.credentials`.
- v2.x added the `runners:` array, kept the legacy fields as a
  back-compat shim, and migrated credentials to
  `/data/runner-credentials/<first-runner-name>/`.
- v3.0.0 dropped the legacy fields entirely. Only the `runners:` array
  works now. The credential-migration block in the launch script is
  retained as a defensive no-op for the (unlikely) v1.x → v3.0 direct
  upgrade path.

If you encounter a v1.x-style options.json in the wild, bring it to
v2.x first (which auto-migrates), then v3.0. Or just configure
`runners:` directly and paste a fresh registration token for each.

## Architecture targeting

`config.yaml` lists `amd64` only. The Dockerfile downloads
`actions-runner-linux-x64-${VERSION}.tar.gz` (hard-coded amd64). For
multi-arch support, both the arch list and the tarball URL need to
template on `${BUILD_ARCH}`. Not currently a priority.

## Memory

Project context worth preserving lives in the project-scoped Claude
memory directory (`<your-claude-config>/projects/<project-slug>/memory/`).
Entries cover the multi-tenant pivot, the compute-budget motivation,
persistence invariants, etc. Update as the project evolves.
