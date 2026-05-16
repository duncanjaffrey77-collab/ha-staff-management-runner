# GitHub Actions Runner (multi-repo)

Self-hosted GitHub Actions runner add-on for HAOS. One add-on instance
serves any number of repos — each runner registers against its own
repo and works independently.

## What it does

- Downloads the [official GitHub Actions runner](https://github.com/actions/runner)
  binary (pinned in `Dockerfile`, `GITHUB_RUNNER_VERSION`).
- Pre-installs Node.js and `wrangler@4` so workflows don't pay cold-start
  cost on every run.
- For each entry in the **Runners** list, registers a runner against the
  named repo and starts it.
- Persists each runner's credentials to `/data/runner-credentials/<name>/`
  so Stop/Start, HA OS reboots, and add-on Updates all reuse them — no
  fresh registration token needed except on first install of each runner.

## Configuration

The **Runners** field is a list. Each entry is one runner:

| Field | Notes |
|---|---|
| `name` | Display name in GitHub's Runners list. Must be unique within this add-on. |
| `repo` | `owner/repo` — any repo where you have admin access |
| `labels` | Comma-separated. Workflows match via `runs-on:`. Keep `self-hosted,linux,x64` at minimum. |
| `registration_token` | One-time token from GitHub. Only needed on first start of this entry. |

`workdir_base` (default `/data/_work`) — parent for each runner's job
work directory (`/data/_work/<name>/`).

### Getting a registration token

1. Open the target repo → **Settings → Actions → Runners**.
2. **New self-hosted runner** → Linux.
3. Find the `./config.sh --url <URL> --token <TOKEN>` line — copy the
   value after `--token`. (You do **not** need to run the commands on
   the page; this add-on does the equivalent.)
4. Paste into the runner entry's `registration_token` field. Save.

⏱️ **Tokens expire ~1 hour after creation.** Generate one immediately
before pasting.

## When IS a fresh token needed?

| What happens | Fresh token? |
|---|---|
| HA OS reboots | No — `/data/runner-credentials/<name>/` survives |
| Add-on Restart / Stop+Start | No |
| Add-on **Update** (image rebuild) | No |
| Add-on Uninstall + Reinstall | **Yes** — uninstall wipes `/data` |
| You delete the runner in GitHub's UI | **Yes** for that specific runner — credentials revoked server-side |
| You add a NEW entry to the `runners` list | **Yes** for the new entry — no credentials yet |

## v1.x → v2.x upgrade

If you're upgrading from v1.x:
1. Your existing config (single runner via the legacy fields
   `github_repo`, `github_runner_token`, `runner_name`, `runner_labels`)
   keeps working. The launch script synthesises a single-element runners
   list from them on every start.
2. To add a second runner, populate the `runners` list with at least the
   existing runner (copy its name/repo/labels — token can be blank if
   it's the runner already registered). Then add the new entry.
3. The launch script migrates your v1.x credentials from
   `/data/runner-credentials/.credentials` to
   `/data/runner-credentials/<first-runner-name>/.credentials` on first
   v2.x start — no token re-paste required.
4. The legacy fields will be removed in v3.0.0.

## Logs

Settings → Add-ons → this add-on → **Log** tab. Lines are prefixed with
the runner name like `[haos-nuc-foo] Starting runner...` so you can
distinguish each runner's output.

## Security

- Registration tokens are masked + encrypted at rest (the `password`
  schema type).
- Each runner runs inside the add-on container, not as root on the host.
- Each runner's long-lived credential is **scoped to that one runner**
  — not a user PAT.
- **Any workflow that matches a runner's labels will execute on your
  machine.** For private single-developer repos this is fine. For
  public repos, configure GitHub → Settings → Actions → "Require
  approval for outside collaborators" before pointing a public repo at
  this runner.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `[<name>] Registration failed.` | Token expired (>1h) or wrong repo. Regenerate, paste, restart. |
| Multiple runners but only one shows up in GitHub | Check the Log tab for per-runner errors — output is prefixed `[<name>]`. |
| `[<name>] No persisted credentials and no registration token` | First start of a new entry needs a token. Add one and Save. |
| Disk usage growing | Each runner's `/data/_work/<name>/` accumulates workflow artifacts. Stop add-on, clean up manually, restart. |
| Slow rebuild after Update | First container start after Update re-creates `/opt/runners/<name>/` per entry (~600MB each). Subsequent restarts are fast. |
