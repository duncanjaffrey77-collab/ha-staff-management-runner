# Repo context (read first)

This repo is a **standalone Home Assistant OS custom add-on** that runs
a self-hosted GitHub Actions runner. It is no longer a downstream copy
of files from another repo — develop the add-on here directly.

- **GitHub:** https://github.com/duncanjaffrey77-collab/ha-staff-management-runner
- **Default target repo for the runner:** `duncanjaffrey77-collab/staff-management`
  (set via the `github_repo` option; can be pointed elsewhere)

## Why this add-on exists

The `staff-management` repo exhausted its GitHub-hosted Actions
compute budget. Hosted runner minutes for that repo's CI/deploy
workflows are no longer affordable. A self-hosted runner on the HAOS
machine consumes **zero** hosted minutes — workflows targeting
`runs-on: [self-hosted, linux, x64, haos]` execute on the user's
hardware.

The add-on is on the critical path: while `staff-management`'s workflows
are flipped to self-hosted, every minute the runner is offline is a
minute of CI queueing. Reliability matters more than feature breadth.

## Layout

```
.
├── README.md                  ← user-facing install + ops guide
├── CLAUDE.md                  ← this file
├── repository.yaml            ← HAOS add-on repo manifest (top-level)
└── github-runner/             ← the add-on package
    ├── Dockerfile             ← pinned runner version + Node + wrangler
    ├── README.md              ← in-UI documentation tab
    ├── config.yaml            ← add-on metadata + options schema
    ├── translations/en.yaml   ← Options form UI labels
    └── rootfs/etc/services.d/runner/run   ← s6-overlay launch script (must be 0755)
```

## Common workflows

### Release a new version

1. Make the change (Dockerfile, config schema, launch script, etc).
2. Bump `version:` in [github-runner/config.yaml](github-runner/config.yaml)
   (semver — patch for bug fixes, minor for features, major for breaking
   schema changes that need users to re-enter Options values).
3. Commit + push to `main`.
4. Tag: `git tag vX.Y.Z && git push --tags`.
5. In HA: Add-on Store → ⋮ → Reload. The add-on tile gains an **Update**
   button.

HA pulls a new image only when `version:` in `config.yaml` changes. If
you forget to bump it, users will not see the update — this is the
single most common shipping mistake.

### Bump the runner binary version

GitHub releases new runners every ~2 weeks.

1. Check https://github.com/actions/runner/releases for the latest tag.
2. Edit [github-runner/Dockerfile](github-runner/Dockerfile) — bump
   `ARG GITHUB_RUNNER_VERSION="..."`.
3. Bump `version:` in `config.yaml` (this is a real version change for
   end users; users will need a fresh registration token because image
   rebuild wipes `/opt/runner/.credentials`).
4. Commit + push + tag.

### Fix a runtime bug

Repro inside the container if possible — HA → Settings → Add-ons →
this add-on → ⋮ → Web terminal (if exposed) or the Terminal add-on can
exec into the container. Then make the fix here, bump version, push.

## Things to **not** do

- **Don't commit the registration token, a PAT, or `.credentials/`.**
  The runner's long-lived credential lives inside the add-on container
  on the host, not in this repo. Nothing sensitive should ever land
  here.
- **Don't rename `github-runner/`.** The HA Supervisor uses that
  directory name as the add-on slug-context. Renaming forces a full
  uninstall/reinstall on the HA side.
- **Don't edit `repository.yaml` `url:`.** It must match this repo's
  origin URL. Changing it breaks the HA add-on store lookup.
- **Don't add CI workflows in this repo that target `ubuntu-latest`.**
  This repo's purpose is to *eliminate* hosted runner usage. Any CI
  here either targets the self-hosted runner (after the add-on is
  installed and verified) or doesn't exist at all.
- **Don't reintroduce a deregister-on-stop trap to the launch script.**
  The original behavior wiped credentials on every Stop and forced a
  fresh token on every Start. The current script intentionally skips
  deregistration — `config.sh remove` is only needed at uninstall time,
  and the user can do it manually in the GitHub UI if they want.

## File mode caveat (Windows)

The s6 launch script
[github-runner/rootfs/etc/services.d/runner/run](github-runner/rootfs/etc/services.d/runner/run)
**must be mode 0755** in the published artifact — s6-overlay refuses
to run a non-executable service script. Git on Windows stores the
executable bit in the index even though the working tree may report
`-rw-`. Verify with:

```bash
git ls-files --stage github-runner/rootfs/etc/services.d/runner/run
# expect:  100755 <sha> 0  github-runner/rootfs/etc/services.d/runner/run
```

If it shows `100644`, fix before pushing:

```bash
git update-index --chmod=+x github-runner/rootfs/etc/services.d/runner/run
git commit -m "fix: mark run script executable"
git push
```

## Runner credential lifecycle (important context)

**v1.0.3+:** The launch script at [run](github-runner/rootfs/etc/services.d/runner/run)
mirrors the runner's three credential files (`.credentials`,
`.credentials_rsaparams`, `.runner`) to `/data/runner-credentials/`
after first-time registration, and restores them from there on every
subsequent start. This means:

- **A registration token is only needed at first install** — never
  again unless the user uninstalls the add-on or manually removes the
  runner in the GitHub UI.
- HA OS reboots, Stop/Start, even add-on **Update** (image rebuild)
  all work with **no fresh token** required — because `/data` persists
  across container recreation.
- The trade-off: when the user clicks **Uninstall**, the runner
  remains registered on the GitHub side as "Offline" until manually
  removed. This is intentional and documented in the README.

**Why this is in /data, not /opt:** HA Supervisor recreates the add-on
container on Stop/Start (not just stops/starts the same one). Anything
in the writable container layer (including `/opt/runner/`) is wiped on
recreation. Only the `/data` mount survives. v1.0.2 incorrectly
assumed `/opt/runner/.credentials` would persist; it doesn't.

If a future change reintroduces deregister-on-stop, the persistence
layer at `/data/runner-credentials/` becomes stale and the next start
will succeed registration with the stale credentials but then fail to
authenticate to GitHub (server has revoked them). Don't reintroduce
deregister-on-stop without ALSO clearing `/data/runner-credentials/`
in the cleanup path.

## Architecture targeting

`config.yaml` currently lists only `amd64`. The Dockerfile downloads
`actions-runner-linux-x64-${VERSION}.tar.gz` — also hard-coded for
amd64. If multi-arch support is ever needed (Raspberry Pi, etc.),
both the arch list and the tarball URL need to template on
`${BUILD_ARCH}`. Not a priority.

## Memory

Project context worth preserving lives in
`C:\Users\dunca\.claude\projects\C--Users-dunca-GitHub-ha-staff-management-runner\memory\`.
Current entries explain the standalone status and the budget-driven
motivation. Update as the project evolves.
