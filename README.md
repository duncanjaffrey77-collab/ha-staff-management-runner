# Home Assistant Add-on: GitHub Actions Runner

A Home Assistant OS custom add-on that runs a self-hosted GitHub Actions
runner on your HAOS machine (NUC, mini PC, server, etc.). Workflows
that target this runner consume **zero** GitHub-hosted runner minutes —
they execute on your hardware instead.

Built for the `duncanjaffrey77-collab/staff-management` repo by default,
but the target repo is a configurable option — point it at any repo
where you have admin access to generate a runner registration token.

## Install

### 1. Add this repository to HAOS

In Home Assistant:

1. **Settings → Add-ons → Add-on Store**
2. Top-right **⋮ menu → Repositories**
3. Paste:
   ```
   https://github.com/duncanjaffrey77-collab/ha-staff-management-runner
   ```
4. **Add → Close**
5. Scroll the store — a new section **Staff Management — Self-hosted CI**
   appears with **GitHub Actions Runner — staff-management** inside.

### 2. Install the add-on

Click the tile → **Install**. First install takes a few minutes — it
downloads the official runner binary, Node.js, and `wrangler` into the
add-on image.

### 3. Get a registration token

In the GitHub repo you want the runner to serve:

1. **Settings → Actions → Runners → New self-hosted runner**
2. Pick **Linux** as the platform.
3. On the displayed `./config.sh --url <URL> --token <TOKEN>` line, copy
   the value after `--token`. You do **not** need to run any of the
   commands on that page — this add-on does the equivalent.

⏱️ **Tokens expire ~1 hour after creation.** If you don't use it
promptly, regenerate it.

### 4. Configure and start

1. Add-on page → **Configuration** tab.
2. Paste the token into **Runner registration token**.
3. (Optional) change `github_repo`, `runner_name`, `runner_labels`,
   `runner_workdir` if you don't want the defaults.
4. **Save**.
5. **Info** tab → **Start**.
6. **Log** tab — you should see:
   ```
   √ Runner successfully added
   √ Runner connection is good
   √ Settings Saved.
   Listening for Jobs
   ```
7. Verify in GitHub: **Settings → Actions → Runners** shows `haos-nuc`
   with a green **Idle** dot.

### 5. Route workflows to it

Edit any workflow in your repo to use the runner's labels:

```yaml
# before
runs-on: ubuntu-latest

# after
runs-on: [self-hosted, linux, x64, haos]
```

**Don't flip this until step 4 shows green.** A workflow that targets
the runner before the runner is online will queue indefinitely (up to
24h) before timing out — there is no automatic fallback to hosted
runners.

## Configuration options

| Option | Default | Notes |
|---|---|---|
| `github_repo` | `duncanjaffrey77-collab/staff-management` | `owner/repo` — any repo where you have admin |
| `github_runner_token` | _(required on first install)_ | Registration token. Expires ~1h after creation |
| `runner_name` | `haos-nuc` | Display name in the GitHub Runners list |
| `runner_labels` | `self-hosted,linux,x64,haos` | Comma-separated. Workflows match via `runs-on:` |
| `runner_workdir` | `/data/_work` | Persistent across add-on restarts |

The token is only required for **first-time registration**. After that,
the runner's long-lived credential lives at `/opt/runner/.credentials`
inside the container, and Stop/Start cycles reuse it — no fresh token
needed. A new token is only required when:

- Installing for the first time
- After an add-on **Update** (image rebuilds wipe the container layer)
- After manually removing the runner in the GitHub UI

## Restart behavior

| What happens | Needs a fresh token? |
|---|---|
| HA OS reboots, runner restarts automatically | No — credentials persist in the container |
| You click **Restart** on the add-on | No — same |
| You click **Stop** then **Start** | No — same |
| You click **Update** on a new add-on version | **Yes** — image rebuild |
| You uninstall + reinstall | **Yes** |
| You manually delete the runner in GitHub UI | **Yes** — credential is invalidated server-side |

## Ongoing ops

### Updating the runner version

GitHub releases new runner versions every ~2 weeks
([changelog](https://github.com/actions/runner/releases)).

1. Edit [github-runner/Dockerfile](github-runner/Dockerfile): bump
   `ARG GITHUB_RUNNER_VERSION="..."`.
2. Edit [github-runner/config.yaml](github-runner/config.yaml): bump
   `version:` (semver — e.g. `1.0.0` → `1.0.1`).
3. Commit + push to `main`.
4. In HA: Add-on Store → **⋮ → Reload**. The add-on tile shows an
   **Update** button. Click it.
5. After update, paste a fresh registration token (image rebuild wiped
   the credential) and Start.

Tag the release for traceability:
```bash
git tag v1.0.1 && git push --tags
```

### Logs

Settings → Add-ons → GitHub Actions Runner → **Log** tab.
Live-tails runner output; per-job step output streams here while a
workflow is running.

### Clean removal

1. Add-on page → **Stop**.
2. **Uninstall**.
3. GitHub UI → Settings → Actions → Runners → **⋮ → Remove** on the
   runner entry (the add-on does NOT auto-deregister; if it did, every
   Stop would require a fresh token to restart, which defeats the
   point).

## Security notes

- The registration token is stored encrypted by Supervisor and never
  appears in logs (the run script references `${TOKEN}` only).
- The runner runs inside the add-on container, not as root on the HAOS
  host.
- The runner's long-lived credential is **scoped to one runner** — it
  is not a user PAT and cannot act on your account.
- **Any workflow that matches the runner's labels will execute on your
  machine.** For a private single-developer repo this is fine. For a
  public repo, configure **GitHub → Settings → Actions → "Require
  approval for all outside collaborators"** before flipping any
  workflow to self-hosted.
- Don't commit the registration token to git. Don't paste it in chat
  or screenshots. It's short-lived (~1h) but treat it as a secret.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Log: `Http response code: NotFound from … runner-registration` | Token expired (>1h old) or `github_repo` is wrong. Regenerate token, paste, restart. |
| Log: `Registration failed.` | Token field empty or pasted with leading/trailing whitespace. Re-copy. |
| Runner shows **Idle** in GitHub UI but a workflow stays queued | `runs-on:` labels in the workflow don't match `runner_labels` on the add-on. |
| Disk filling up | `/data/_work` accumulates per-workflow checkouts. Stop add-on → use the Terminal add-on to `rm -rf /usr/share/hassio/addons/data/<slug>/_work/*` → restart. |
| OOM during a Node build | Add `NODE_OPTIONS=--max-old-space-size=4096` to the workflow env. Check HA → Settings → System → Hardware for total RAM. |
| `wrangler` not found | The image installs `wrangler@4` globally. `npx wrangler ...` always resolves. If a workflow `npm ci`s first, the local devDependency takes precedence — that's intended. |

## Repository layout

```
.
├── README.md                  ← this file
├── CLAUDE.md                  ← context for AI assistants working in this repo
├── repository.yaml            ← HAOS add-on repo manifest (top-level)
└── github-runner/             ← the add-on package
    ├── config.yaml            ← metadata + options schema
    ├── Dockerfile             ← image build (pinned runner + Node + wrangler)
    ├── README.md              ← in-UI documentation tab
    ├── translations/en.yaml   ← Options form labels
    └── rootfs/etc/services.d/runner/run   ← s6-overlay launch script
```

## Contributing

This is a single-author project for now, but if it turns out useful to
others, PRs welcome. Keep changes focused on the runner add-on itself —
this repo is not a general-purpose CI runner framework.

When making changes:

1. Bump `version:` in [github-runner/config.yaml](github-runner/config.yaml).
2. If touching the launch script, verify it stays mode `0755`
   (`git ls-files --stage` must show `100755`, not `100644` — s6-overlay
   refuses to run non-executable service scripts).
3. Tag the release: `git tag vX.Y.Z && git push --tags`.
