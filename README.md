# Home Assistant Add-on: GitHub Actions Runner (multi-repo)

A Home Assistant OS custom add-on that runs the official GitHub Actions
runner against **any number of GitHub repositories from one HA install**.
Workflows that target a runner registered by this add-on consume **zero**
GitHub-hosted runner minutes — they execute on your hardware.

Originally built to relieve the `staff-management` repo's CI compute
budget; v2.0.0 generalised it into a multi-tenant tool.

## Install

### 1. Add this repository to HAOS

1. **Settings → Add-ons → Add-on Store**
2. Top-right **⋮ menu → Repositories**
3. Paste:
   ```
   https://github.com/duncanjaffrey77-collab/ha-staff-management-runner
   ```
4. **Add → Close**.
5. Find **GitHub Actions Runner (multi-repo)** in the store.

### 2. Install the add-on

Click the tile → **Install**. First install pulls the Debian base image,
Node 20, wrangler, and the actions/runner binary (~5 minutes total).

### 3. Configure your first runner

Add-on page → **Configuration** tab. The **Runners** field is a list —
add one entry:

```yaml
runners:
  - name: haos-nuc-myrepo
    repo: youruser/yourrepo
    labels: self-hosted,linux,x64,haos
    registration_token: <PASTE_FRESH_TOKEN>
workdir_base: /data/_work
```

To generate a token: target repo → **Settings → Actions → Runners →
New self-hosted runner → Linux** → copy the value after `--token` on
the `./config.sh` line. ⏱️ Tokens expire ~1 hour after creation.

Save → Info → **Start**.

### 4. Verify

Log tab — look for:
```
[<name>] Starting runner...
√ Connected to GitHub
Listening for Jobs
```

GitHub side: target repo → **Settings → Actions → Runners** → your
runner shows as **Idle**.

### 5. Route workflows

In any workflow targeting one of these runners:

```yaml
# before
runs-on: ubuntu-latest

# after
runs-on: [self-hosted, linux, x64, haos]
```

A workflow that targets `runs-on:` labels with no matching runner queues
indefinitely (up to 24h) before timing out. There is no automatic
fallback to hosted runners — see [staff-management's dispatcher pattern](https://github.com/duncanjaffrey77-collab/staff-management/blob/master/.github/workflows/ci.yml)
for one approach.

## Adding more runners

Same Configuration tab — click **+** on the runners list and fill in a
new entry. Each runner needs:
- A unique `name`
- A `repo` (`owner/repo`) where you have admin
- A `labels` string
- A fresh `registration_token` (one-time, for first start only)

Save → the add-on automatically picks up the new entry and registers it
on next restart. Existing runners are untouched.

## Configuration reference

### `runners` (list)

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique runner name (also the directory name in `/data/runner-credentials/`) |
| `repo` | string | `owner/repo` of the target GitHub repository |
| `labels` | string | Comma-separated labels for `runs-on:` matching |
| `registration_token` | password | One-time token from GitHub — only used on first start of this entry |

### Top-level

| Field | Default | Notes |
|---|---|---|
| `workdir_base` | `/data/_work` | Parent dir for each runner's `_work` folder |
| `github_repo` *(legacy)* | _(empty)_ | v1.x compat only; ignored if `runners` is non-empty |
| `github_runner_token` *(legacy)* | _(empty)_ | v1.x compat only |
| `runner_name` *(legacy)* | `haos-nuc` | v1.x compat only |
| `runner_labels` *(legacy)* | `self-hosted,linux,x64,haos` | v1.x compat only |

The legacy fields exist so v1.x → v2.0 upgrades are seamless. New
installs should leave them blank and use the `runners` list. Legacy
fields will be removed in v3.0.0.

## Restart behavior

After first registration, each runner's credentials are mirrored to
`/data/runner-credentials/<name>/` (which survives container recreation).

| What happens | Fresh token? |
|---|---|
| HA OS reboots | No |
| Add-on Restart / Stop+Start | No |
| Add-on **Update** | No (image rebuilds but `/data` persists) |
| Add-on Uninstall + Reinstall | **Yes** |
| Manually delete the runner in GitHub UI | **Yes** for that runner |
| Add a new runner entry | **Yes** for the new entry only |

## v1.x → v2.0 upgrade

Existing v1.x installs upgrade seamlessly:
1. After the Update, the launch script reads your legacy
   `github_repo` / `github_runner_token` / `runner_name` / `runner_labels`
   fields if the new `runners` list is empty.
2. Credentials at `/data/runner-credentials/.credentials` (flat v1.x
   layout) are auto-migrated to
   `/data/runner-credentials/<runner_name>/` on first start. No token
   re-paste required.
3. When you want a second runner, move the existing config from the
   legacy fields into the new `runners` list, then add the new entry.
   The legacy fields can be cleared at that point.

The legacy fields will be removed in v3.0.0. There's no rush; v2.x will
support both indefinitely.

## Updating the runner binary version

GitHub releases new runner versions every ~2 weeks
([changelog](https://github.com/actions/runner/releases)).

1. Edit [github-runner/Dockerfile](github-runner/Dockerfile): bump
   `ARG GITHUB_RUNNER_VERSION="..."`.
2. Edit [github-runner/config.yaml](github-runner/config.yaml): bump
   `version:` (semver).
3. Commit + push to `main`. Tag the release for traceability.
4. In HA: Add-on Store → **⋮ → Reload**. Click Update on the tile.

Image rebuild wipes the `/opt/runners/<name>/` per-instance directories,
but `/data/runner-credentials/<name>/` survives — each runner reuses its
credentials, no fresh tokens needed.

## Logs

Settings → Add-ons → this add-on → **Log** tab. Lines from each runner
are prefixed `[<name>] ...` so you can distinguish them when multiple are
configured.

## Clean removal

1. Add-on → **Stop**.
2. **Uninstall**.
3. GitHub UI → each target repo → Settings → Actions → Runners →
   **⋮ → Remove** for each runner. The add-on does NOT auto-deregister
   on stop (Stop/Start would otherwise need fresh tokens, defeating the
   point).

## Security

- Registration tokens stored encrypted by Supervisor; masked in UI;
  never logged.
- Each runner runs inside the add-on container, not as root on the host.
- Each runner's credential is scoped to its single runner — not a PAT.
- **Any workflow whose `runs-on:` matches a runner's labels will execute
  on your machine.** Private single-developer repos: fine. Public repos:
  configure GitHub → Settings → Actions → "Require approval for outside
  collaborators" before pointing a public repo here.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `[<name>] Registration failed.` | Token expired (>1h) or wrong repo. Regenerate, paste, restart. |
| One runner shows up but another doesn't | Per-runner errors are prefixed `[<name>]` in the Log tab — look for the missing one. |
| All runners offline after Update | Image rebuild wiped `/opt/runners/`. Should auto-recover by restoring from `/data/runner-credentials/<name>/`. If it doesn't, the per-runner credentials may be missing — paste fresh tokens. |
| Disk filling up | Each runner's `/data/_work/<name>/` accumulates. Stop add-on, clean up manually, restart. |

## Repository layout

```
.
├── README.md                  ← this file
├── CLAUDE.md                  ← context for AI assistants
├── repository.yaml            ← HAOS add-on repo manifest
└── github-runner/             ← the add-on package
    ├── config.yaml            ← metadata + options schema
    ├── Dockerfile             ← Debian base + Node + wrangler + runner template
    ├── build.yaml             ← base image pin
    ├── README.md              ← in-UI documentation tab
    ├── translations/en.yaml   ← Options form labels
    └── rootfs/etc/services.d/runner/run   ← multi-runner supervisor (must be 0755)
```
