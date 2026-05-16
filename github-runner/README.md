# GitHub Actions Runner — staff-management

A Home Assistant OS add-on that runs the official GitHub Actions runner
binary against the `staff-management` repository. Workflows targeted at
this runner consume **zero** GitHub-hosted runner minutes — they run on
your NUC instead.

## What it does

- Downloads and installs the [official GitHub Actions runner](https://github.com/actions/runner)
  binary (pinned version — see `Dockerfile` `GITHUB_RUNNER_VERSION`).
- Pre-installs Node.js and `wrangler@4` so workflows don't pay cold-start
  cost on every run.
- Registers the runner against your repo using a one-time registration
  token you paste in the Options tab.
- Cleanly de-registers on Stop (no "Offline" zombies in the GitHub UI).

## Configuration

| Option | Default | Notes |
|---|---|---|
| `github_repo` | `duncanjaffrey77-collab/staff-management` | `owner/repo`. |
| `github_runner_token` | _(required)_ | Registration token from GitHub. **Expires ~1 hour after creation.** |
| `runner_name` | `haos-nuc` | Shown in GitHub → Settings → Actions → Runners. |
| `runner_labels` | `self-hosted,linux,x64,haos` | Workflows match these via `runs-on:`. |
| `runner_workdir` | `/data/_work` | Persistent — survives add-on restart. |

## Getting a registration token

1. Open the GitHub repo → **Settings → Actions → Runners**.
2. Click **New self-hosted runner**.
3. Pick **Linux** as the platform.
4. Look for the `./config.sh --url <URL> --token <TOKEN>` line — copy the
   value after `--token`. (You do **not** need to run the commands on the
   page; this add-on does the equivalent.)
5. Paste it into the **Runner registration token** field on the Options
   tab of this add-on. Save. Start the add-on.

The token is only used once — at registration time. After that the runner
holds its own long-lived credential in `/opt/runner/.credentials`. The
token is only re-needed if the add-on container is rebuilt (e.g. version
bump) **and** the prior credentials don't survive.

## Verifying it works

1. After **Start**, open the **Log** tab. You should see
   `√ Runner successfully added` and then `Listening for Jobs`.
2. Open your GitHub repo → **Settings → Actions → Runners**. The runner
   name (default `haos-nuc`) should appear with a green **Idle** dot.
3. Trigger any workflow whose `runs-on:` matches the runner's labels.

## Updating the runner version

1. Edit `github-runner/Dockerfile`: bump `GITHUB_RUNNER_VERSION`.
2. Edit `github-runner/config.yaml`: bump `version:` (e.g. `1.0.0` → `1.0.1`).
3. Commit + push the add-on repo. HA will offer **Update** on the add-on
   page on the next refresh.

## Logs

Settings → Add-ons → GitHub Actions Runner → **Log** tab. The runner's
job logs (per-step output) stream here while a workflow is running.

## Security

- The registration token is stored encrypted by Supervisor and masked in
  the UI (`schema: password`). It never lands in logs (the run script
  references `${TOKEN}` only, never echoes it).
- The runner runs as the add-on container's user, not root on the host.
- The runner has read-only access to the GitHub repo via its own
  long-lived credential — it does **not** have access to your account.
- **The runner WILL run any workflow that targets its labels.** Treat
  the repo's `master` branch like production: anyone with push access
  to the repo can execute arbitrary code on your NUC. For a private
  single-developer repo this is fine; for a public repo, gate via
  GitHub's "Require approval for first-time contributors" setting.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Log shows `Http response code: NotFound from 'POST … /actions/runner-registration'` | Token expired (>1h old) or wrong `github_repo`. Regenerate token + restart. |
| Log shows `Runner registration failed.` | Token is empty or malformed. Re-copy from the GitHub UI. |
| Runner appears in GitHub UI but workflow stays queued | `runs-on:` labels in the workflow don't match `runner_labels` on the add-on. |
| Disk usage growing | Workflows leave artifacts in `/data/_work/<repo>/<repo>`. Periodically Stop add-on + `rm -rf` via Terminal add-on, or set a `cleanup` step in the workflow. |
