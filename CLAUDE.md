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
    ├── Dockerfile             ← Debian Bookworm + Node 20 + wrangler + libsqlite3 + runner template
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

Each configured runner has paths in multiple layers, by design to balance
backup size against cold-start cost:

| Path | Lifetime | In `/data` (HAOS backup)? | Purpose |
|---|---|---|---|
| `/opt/runner/` | image rebuild | no | Template install — read-only at runtime. Built into the Docker image. |
| `/opt/runners/<name>/` | container layer | no | Per-instance install — `cp -r` from `/opt/runner/` on first start of this entry. Wiped by image rebuild. |
| `/data/runner-credentials/<name>/` | host volume | yes | Persisted `.credentials`, `.credentials_rsaparams`, `.runner` — survives container recreation AND image rebuild. |
| `/work/<name>/` (v3.0.6+) | container layer | **no** | Workflow job working dir — repo checkouts, `_actions`, `_temp`. **Ephemeral**: wiped on container recreation. Pre-v3.0.6 this lived at `/data/_work/<name>/` and bloated HAOS backups. |
| `/data/cache/tools/` (v3.0.6+) | host volume | yes (small) | `RUNNER_TOOL_CACHE` — SDK installs (Flutter, Node, Java) keyed by version. Shared across runners. Persists for fast cold-restart. |
| `/data/cache/<name>/pub/` (v3.0.6+) | host volume | yes (small) | `PUB_CACHE` — Flutter pub cache. Per-runner. |
| `/data/cache/<name>/gradle/` (v3.0.6+) | host volume | yes (small) | `GRADLE_USER_HOME` — Android Gradle cache. Per-runner (REQUIRED — Gradle daemon locks conflict if shared). |
| `/data/cache/<name>/npm/` (v3.0.6+) | host volume | yes (small) | `npm_config_cache` — npm install cache. Per-runner. |

**Boot sequence per runner:**
1. If `/opt/runners/<name>/` is missing → `cp -r /opt/runner /opt/runners/<name>/`.
2. If `/opt/runners/<name>/.credentials` missing AND
   `/data/runner-credentials/<name>/.credentials` exists → restore from
   `/data`.
3. If still no credentials → register using the entry's
   `registration_token`. Mirror credentials to `/data/runner-credentials/<name>/`.
4. Enter the **self-heal loop** wrapping `./run.sh` (see below).

The supervisor `wait`s for all runners; if any exit, the others keep
working. If all exit, the supervisor exits and s6 restarts the whole
service.

### Self-heal loop (v3.0.2+)

Each runner's `./run.sh` is wrapped in a retry loop that detects a
specific class of failure — server-side registration invalidated —
and recovers without manual shell access to `/data`.

**Trigger patterns in `./run.sh` stdout:**
- `registration has been deleted from the server` (GitHub's 14-day
  inactivity sweep)
- `System:PublicAccess … not authorized` (credentials mismatch with
  server-side state)

**On detection:**
1. Delete local credentials and `/data/runner-credentials/<name>/`.
2. Attempt re-registration with the `registration_token` from HA Options.
3. Token still valid (rare — single-use) → loop back into `./run.sh`,
   full self-heal.
4. Token already consumed → exit cleanly with a log message pointing
   at HA Options. The stale creds are still gone, so the next manual
   restart-with-fresh-token recovers in **two steps** (not five).

**Manual recovery flow after self-heal exits cleanly:**
1. Generate a fresh registration token from the affected repo's
   Settings → Actions → Runners → New self-hosted runner page.
2. Paste into the entry's `registration_token:` in HA Options, restart
   the add-on.

Retry cap: 3 attempts per runner per service-restart, to prevent infinite
loops on permanently bad config.

**Non-auth exits** (clean shutdown, crash, etc.) pass through the loop
unchanged — supervisor `wait` notices and s6 restarts the service group.

## Known issues / sharp edges

### "One random runner goes stale on every add-on restart"

**Observation:** Every time the add-on container is restarted (HA
Update, manual Restart, image rebuild after a `version:` bump), exactly
one of the configured runners has its server-side registration silently
invalidated. The runner connects with its restored-from-`/data`
credentials and fails with `PublicAccess … not authorized`. Different
runner each time; not deterministic which.

Observed during the 2026-05-18 → 2026-05-20 stabilisation session
across all three runners (haos-nuc, haos-nuc-surgical, haos-nuc-qcall).
Self-heal (v3.0.2+) makes the symptom much less painful — failed runner
cleans up its own creds and exits with a clear actionable error — but
doesn't address root cause.

**Hypotheses (untested):**
1. **Parallel reconnect race.** All runners restart simultaneously and
   reconnect to GitHub within milliseconds. GitHub may treat
   near-simultaneous re-connects from the same install as suspicious
   and invalidate one registration.
2. **Ungraceful shutdown corrupting state.** When the container is
   killed, s6 sends SIGTERM to the parent `run` script. The script is
   blocked in `wait`, and its background subshells running `./run.sh`
   don't receive SIGTERM directly. They die when bash exits, but
   without a chance to flush rotated credentials. The persisted
   `.credentials` is then stale relative to GitHub's view.
3. **Token rotation mid-flight.** actions/runner rotates its auth
   token internally on a schedule. If the container is killed during
   a rotation, the on-disk credential lags the server-side state.

Likely fix candidates:
- Trap SIGTERM in the parent `run` script and forward to all
  background subshells. Give each ~30s to terminate cleanly before
  the container exits.
- Add a small stagger (~5s) between starting each runner so they
  don't reconnect in parallel.

### Single-use registration tokens — recovery requires fresh tokens

Registration tokens are single-use and expire in ~1h. The launch
script consumes the token on first successful registration. If
`./config.sh` is re-invoked (e.g. by the self-heal loop), the same
token won't work a second time. This is GitHub's design, not the
add-on's. Practical implication: the self-heal loop's re-registration
step almost always fails (token already consumed); its real value is
the credential cleanup, which makes the next manual recovery a
2-step affair.

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
- v3.0.1 bumped the actions/runner binary pin 2.319.1 → 2.334.0.
  Reason: 2.327.0+ adds node24 manifest support; without it, fresh
  installs crash on `actions/cache@v5` and similar transitive deps
  used by `subosito/flutter-action@v2`.
- v3.0.2 added the self-heal loop wrapping `./run.sh` — see the
  "Self-heal loop" subsection above.
- v3.0.3 baked `libsqlite3-0` into the runner image. Flutter projects
  using `drift` (Dart SQLite ORM) dlopen `libsqlite3.so.0` at test
  time; minimal Bookworm doesn't ship it.
- v3.0.5 added `libsqlite3-dev` so the unversioned `libsqlite3.so`
  symlink exists. Dart's `sqlite3` package calls
  `DynamicLibrary.open('libsqlite3.so')` (unversioned), not `.so.0`.
- v3.0.6 split the runner state model into a hybrid layout to keep
  HAOS backups small without losing warm-restart speed:
  - Work tree base moves from `/data/_work/<name>/` to `/work/<name>/`
    (container layer, NOT in `/data`, NOT in HAOS backups).
  - SDK + dep caches (`RUNNER_TOOL_CACHE`, `PUB_CACHE`,
    `GRADLE_USER_HOME`, `npm_config_cache`) redirect to
    `/data/cache/...` so they persist across container recreations.
  - On upgrade: the launch script auto-patches each runner's `.runner`
    JSON to point at the new workFolder. Existing `/data/_work/` from
    pre-v3.0.6 is left in place as an orphan — user can `rm -rf` to
    reclaim space (~10GB in the canonical install).
  - The `workdir_base` HA Option still honoured; the legacy default
    value `/data/_work` is treated as "use new default" so users who
    never explicitly set it auto-migrate.

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
