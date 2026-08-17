# dotfiles

Personal [chezmoi](https://www.chezmoi.io/)-managed dotfiles for provisioning
a consistent development environment on macOS and WSL2 Ubuntu. GitHub
account: `gravytrain-willisa`.

It provisions:
- zsh + oh-my-zsh + starship
- Homebrew/Linuxbrew packages
- SDKMAN Java/Gradle/Maven
- nvm Node versions and global npm tools
- pyenv Python versions and pip tools
- Docker image pulls (including ECR)
- 1Password-backed SSH agent, AWS credentials (aws-vault), and API-token
  workflows
- Git config (identity, commit signing), AWS config, SSH config
- Shell aliases/helper functions, repo cloning, and update tooling

For the full ordered setup sequence and the rationale behind each step, see
[`docs/setup-sequence.md`](docs/setup-sequence.md) — this README covers the
day-to-day path: install, customize, common commands, and troubleshooting.

## Contents

- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Manual setup checklist](#manual-setup-checklist)
- [Common commands](#common-commands)
- [How to customize](#how-to-customize)
- [Important behavior](#important-behavior)
- [Troubleshooting](#troubleshooting)
- [Development / testing this repo](#development--testing-this-repo)
- [Repo layout](#repo-layout)
- [WSL2 disk space management](#wsl2-disk-space-management-windows-side-manual)
- [Notes](#notes)
- [Maintenance notes for agents](#maintenance-notes-for-agents)

## Prerequisites

- macOS or WSL2 Ubuntu.
- GitHub access to this **private** repository — the Quick start command
  below will fail on the initial `git clone` without it.
- `curl` and `git` available on the machine already. They are usually
  present on macOS and many Ubuntu/WSL images; install them first if the
  bootstrap command fails before chezmoi starts.
- Optional but expected: Docker Desktop and the 1Password desktop app (see
  the manual checklist below for what each unlocks).

## Quick start

On a new macOS or WSL2 Ubuntu machine:

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- -b "$HOME/.local/bin" init --apply gravytrain-willisa/dotfiles && exec zsh
```

That single command runs every automatable setup step — it pulls the repo
and runs the chezmoi scripts in order, including getting itself and `zsh`
onto Homebrew/Linuxbrew by the end (see
[`docs/setup-sequence.md`](docs/setup-sequence.md) items 1-3 and 20,
"Shell"/"oh-my-zsh"/"Homebrew"/"Handoff", for why those two are
special-cased on a fresh machine). The trailing `&& exec zsh` picks up the
brew-managed shell for you — whether the terminal started in some other
shell (switches into zsh) or was already in zsh (replaces it with a fresh
one, reloading every rc file) — so there's no separate "open a new shell"
step to remember.

It asks for your sudo password once, near the start, then grants passwordless
sudo for the rest of the apply's bootstrap commands (see
[`docs/setup-sequence.md`](docs/setup-sequence.md) item 0, "Passwordless sudo
for the bootstrap commands") — so leave the terminal on that prompt rather
than walking away expecting an unattended install.

Everything after `init` is passed straight through to `chezmoi init`, so to
install and initialize from a specific branch instead of `main` right from
the start, add `--branch`:

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- -b "$HOME/.local/bin" init --apply --branch my-branch gravytrain-willisa/dotfiles && exec zsh
```

See "Testing a pushed branch before merging to main" below for the caveats
that come with running against a non-`main` branch.

A handful of steps genuinely can't be automated (GUI toggles, Windows-side
settings, one-time account setup) — work through the manual checklist below
next.

## Manual setup checklist

Chezmoi can't do these for you. Item numbers below refer to
[`docs/setup-sequence.md`](docs/setup-sequence.md) for the full rationale.

### WSL2 / Windows

| Step | Action |
|---|---|
| Windows Terminal | Settings → Startup → Default profile → your WSL Ubuntu distro (item 5, "Terminal prompt") |
| Docker | Install Docker Desktop and enable WSL2 integration, if you want Docker image pulls (item 14, "Docker images") |
| 1Password SSH Agent | 1Password → Settings → Developer → **Use the SSH Agent** (item 15, "1Password SSH agent relay") |
| 1Password Windows Hello | 1Password → Settings → Security → **Unlock using Windows Hello** — required before the CLI integration toggle works (item 17, "aws-vault") |
| 1Password CLI integration | 1Password → Settings → Developer → **Integrate with 1Password CLI** (item 17, "aws-vault") |
| PowerShell execution policy | Only if you see "running scripts is disabled": `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` — a genuine security control, not a GUI toggle (item 5, "Terminal prompt") |
| aws-vault env vars | Set `AWS_VAULT_BACKEND`, `AWS_VAULT_OP_VAULT_ID`, `AWS_VAULT_OP_DESKTOP_ACCOUNT_ID` as persistent Windows user env vars — required before `aws-vault`/`aws-login` work at all (item 17, "aws-vault") |
| AWS region (native Windows tools only) | Set `AWS_REGION`/`AWS_DEFAULT_REGION` as persistent Windows user env vars — only needed if you run `aws-vault.exe`/`aws.exe` directly outside WSL (item 19, "AWS default region") |

### macOS

| Step | Action |
|---|---|
| Docker | Install Docker Desktop, if you want Docker image pulls (item 14, "Docker images") |
| 1Password SSH Agent | 1Password → Settings → Developer → **Use the SSH Agent** (item 15, "1Password SSH agent relay") |
| 1Password CLI integration | 1Password → Settings → Developer → **Integrate with 1Password CLI** — required for `op-login` and 1Password-backed `aws-vault` (item 17, "aws-vault") |
| aws-vault env vars | Set `1password.op_vault_id`/`1password.op_account_id` in [`.chezmoidata/dotfiles.yaml`](.chezmoidata/dotfiles.yaml), then `chezmoi apply` — required before `aws-vault`/`aws-login` work at all, same as WSL (item 17, "aws-vault") |

### Optional, either platform

| Step | Action |
|---|---|
| `op-login` API keys | Create 1Password items for Anthropic/OpenAI/Snyk (or whatever you add to `op-env-vars.txt`) (item 18, "Env vars from 1Password") |
| Git commit signing | Set `git.signing_key` in `.chezmoidata/dotfiles.yaml` to your SSH key's public half (item 23, "Git config") |

## Common commands

| Command | Purpose |
|---|---|
| `chezmoi diff` | Preview what `chezmoi apply` would change — renders every template, runs no scripts |
| `chezmoi apply -v` | Apply for real, printing each script's output |
| `chezmoi update` | `git pull` the source dir *then* `chezmoi apply` — use this, not plain `apply`, to pick up changes made/pushed since the source dir was last cloned |
| `update-all` | Update chezmoi, brew/apt, npm globals, and SDKMAN metadata in one go (runs `chezmoi update`, so it also pulls) |
| `op-login` | Export 1Password-backed env vars (`op-env-vars.txt`) into this shell |
| `aws-login <profile>` | Export AWS credentials for `<profile>` (12h); also refreshes CodeArtifact + ECR auth |
| `docker-update-images` | Re-pull the current tag of every locally present Docker image |
| `docker-prune-all` | `docker system prune --volumes -f` |
| `git-fetch-all` | `git fetch --all --tags --force --prune --prune-tags` in every repo under `~/projects` |
| `git-cleanup-all` | `git gc --aggressive` in every repo under `~/projects` |
| `gradle-refresh-dependencies` | Clean Gradle build with dependencies refreshed, skipping checkstyle/spotbugs tasks the project doesn't have |
| `gradle-show-dependency-updates` | `./gradlew dependencyUpdates` |
| `npm-recreate-package-lock` | Recreate `package-lock.json` and `node_modules` from scratch |
| `cd-projects` | `cd ~/projects` |
| `kgm-connect-to-jump-server <stage> <ip>` / `triton-connect-to-jump-server <stage> <ip>` | SSH tunnel to that system's jump box (needs `aws-login` first) |
| `help` | Show this list from inside the shell, plus your configured direct-server aliases |

**`apply` renders whatever is already checked out in the source dir
(`~/.local/share/chezmoi`) — it never fetches from GitHub.** If a change was
made after that checkout was last cloned or pulled (by you, on another
machine, or merged by someone else), plain `chezmoi apply`/`diff` won't see
it until the source dir is updated — either `chezmoi update`, or `git pull`
inside `~/.local/share/chezmoi` directly (needed instead of `chezmoi update`
when it's a symlink to a local working copy, per "Testing against a local,
not-yet-pushed copy" below — that copy has no upstream to pull from other
than the one you're already editing).

## How to customize

| To change | Edit |
|---|---|
| Shell choice, git identity, GitHub username, AWS region, RDS config, server aliases | [`.chezmoidata/dotfiles.yaml`](.chezmoidata/dotfiles.yaml) |
| Brew packages (plain formula, or `user/tap/formula` for a third-party tap) | [`dot_config/dotfiles/brew-packages.txt`](dot_config/dotfiles/brew-packages.txt) |
| SDKMAN candidates (Java/Gradle/Maven) | [`dot_config/dotfiles/sdkman-packages.txt`](dot_config/dotfiles/sdkman-packages.txt) |
| Node versions | [`dot_config/dotfiles/nvm-versions.txt`](dot_config/dotfiles/nvm-versions.txt) |
| Global npm packages | [`dot_config/dotfiles/node-globals.txt`](dot_config/dotfiles/node-globals.txt) |
| Python versions | [`dot_config/dotfiles/pyenv-versions.txt`](dot_config/dotfiles/pyenv-versions.txt) |
| Global pip packages | [`dot_config/dotfiles/pip-packages.txt`](dot_config/dotfiles/pip-packages.txt) |
| Docker images to pull | [`dot_config/dotfiles/docker-images.txt`](dot_config/dotfiles/docker-images.txt) |
| Repos to clone into `~/projects` | [`dot_config/dotfiles/git-repos.txt`](dot_config/dotfiles/git-repos.txt) |
| Beanstalk → GitHub migration pairs | [`dot_config/dotfiles/beanstalk-github-migrations.txt`](dot_config/dotfiles/beanstalk-github-migrations.txt) |
| 1Password-sourced env vars | [`dot_config/dotfiles/op-env-vars.txt`](dot_config/dotfiles/op-env-vars.txt) |
| Shell aliases/functions | [`dot_zshrc.tmpl`](dot_zshrc.tmpl) — keep `help`'s static list in sync (see [CLAUDE.md](CLAUDE.md)) |
| Git config | [`dot_gitconfig.tmpl`](dot_gitconfig.tmpl) |
| SSH config | [`private_dot_ssh/config.tmpl`](private_dot_ssh/config.tmpl) |
| AWS config file (`~/.aws/config`) | [`dot_aws/config.tmpl`](dot_aws/config.tmpl) |

After editing a manifest or template:

```bash
chezmoi apply
```

Only the script tied to the file you changed re-runs — see "Important
behavior" below for exactly when.

All manifests under `dot_config/dotfiles/*.txt` share the same convention:
one entry per line, blank lines and `#`-comments ignored, **additive-only** —
removing a line stops future installs but never uninstalls anything already
present.

## Important behavior

- Manifests are additive-only — removing a line never uninstalls/removes
  anything already present; `chezmoi apply` never runs `sdk uninstall`/`nvm
  uninstall`/`pyenv uninstall`/`brew uninstall`/`pip uninstall`/`docker rmi`
  on your behalf.
- `run_once_*` scripts run once per machine ever (state tracked by chezmoi,
  not the repo) — force a re-run after editing one with `chezmoi state
  delete-bucket --bucket=scriptState`.
- `run_onchange_*` scripts re-run automatically whenever the manifest they're
  hashed against changes — no extra step needed.
- `~/.zshrc`, `~/.gitconfig`, `~/.ssh/config`, and `~/.aws/config` are fully
  chezmoi-owned and get overwritten on `chezmoi apply` — anything added to
  them by hand outside this repo is lost on the next apply (see
  [`docs/setup-sequence.md`](docs/setup-sequence.md) item 19, "AWS default
  region", for `~/.aws/config` specifically, including how to fold existing
  content back in first).
- The Windows PowerShell profile is append-only, not chezmoi-owned — only one
  line gets injected; the rest of your profile is left untouched.
- WSL-exported shell env vars do **not** reach Windows-native binaries
  (`aws-vault.exe`, `aws.exe`, `op.exe`) — those need a persistent Windows
  user env var instead (see the manual checklist above).
- Some changes (a new `PATH` entry, a new shell function) require a new
  terminal or `exec zsh` to take effect.
- `git-repos.txt`, `docker-images.txt`, and similar manifests reflect real
  internal infrastructure — **keep this repo private**.

## Troubleshooting

**`aws-vault.exe` not found in a new shell right after install** — WSL only
re-reads the Windows `PATH` it inherits via interop at session start. Close
the current WSL terminal, run this from Windows (PowerShell or Command
Prompt, not WSL), then open a new WSL terminal:
```powershell
wsl --shutdown
```

**`aws-vault exec <profile> -- aws ...` says `exec: "aws": executable file
not found in %PATH%`** — under WSL, `aws-vault.exe` is a genuine Windows
process, so the `aws` it spawns is looked up via the Windows `%PATH%`, not
WSL's Linux `PATH` — `which aws` succeeding inside WSL doesn't mean anything
here. The setup installs the Windows AWS CLI for exactly this reason; if it
was just installed, restart WSL (`wsl --shutdown` from Windows, then open a
new terminal) so the new Windows `PATH` entry is picked up. See
`docs/setup-sequence.md` item 17, "aws-vault".

**`aws-vault` says `environment variable unset or empty: "OP_VAULT_ID"`** —
happens on **macOS too**, not just WSL: `op-desktop` hard-requires a vault id
(and an account id) with no fallback. Set `1password.op_vault_id`/`1password.op_account_id`
in `.chezmoidata/dotfiles.yaml` (find the values with `op vault list` / `op
account list --format=json` — use `account_uuid` from the JSON, not the
`USER ID` column from the plain table), then `chezmoi apply` and open a new
terminal. See `docs/setup-sequence.md` item 17, "aws-vault".

**1Password prompts on every single `aws-vault`/`op-login` call** — expected;
each is a fresh process with its own fresh authorization. `aws-login`/
`op-login` batch what they can into one prompt per shell session, but
there's no 1Password setting to suppress this further for third-party
integrations — see `docs/setup-sequence.md` items 17-18 ("aws-vault", "Env
vars from 1Password").

**PowerShell says "running scripts is disabled on this system"** — the
profile that adds the starship prompt is blocked by the default execution
policy; fix once per PowerShell binary:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Cloning repos fails with `Permission denied (publickey)`, or the script
aborts saying "No SSH key available from the agent"** — the 1Password SSH
agent isn't reachable. Almost always the one manual step that can't be
scripted: 1Password desktop → Settings → Developer → **"Use the SSH Agent"**.
Switch it on, then re-run `chezmoi apply -v` — a script that failed isn't
recorded as having completed, so chezmoi retries it without needing to clear
script state.
Check what the agent is actually offering with:
```bash
ssh-add -l
```
If that says "Could not open a connection to your authentication agent" in a
*new* WSL terminal, the relay itself didn't start — confirm
`~/.local/bin/npiperelay.exe` exists and `socat` is installed (both come from
`docs/setup-sequence.md` item 15).

**`ssh-add -l` says "error fetching identities: communication with agent
failed"** even though `~/.1password/agent.sock` exists, `socat` is running,
and `npiperelay.exe` is present — different failure from the one above: the
relay *started* but can't actually exec the Windows binary. Confirm with:
```bash
cat /proc/sys/fs/binfmt_misc/WSLInterop
```
If that file doesn't exist (or reports disabled), WSL's interop registration
for launching Windows `.exe`s from WSL never came up on this boot — a
one-off race, seen in particular with `systemd=true` set in `wsl.conf`,
distinct from the WSL/1Password config itself. Fix with a full restart of
the WSL VM (not just a new terminal), from Windows (PowerShell or Command
Prompt, not WSL):
```powershell
wsl --shutdown
```
then reopen WSL and retry `ssh-add -l`.

**An IDE's own git integration (IntelliJ, VS Code, etc.) gets `Permission
denied (publickey)` on a WSL project even though `git pull`/`git push` work
fine from a terminal** — the IDE invokes `git` via `wsl.exe -d <distro> -e
git ...` directly, which execs straight into the target command rather than
going through a login shell, so it skips PAM entirely — confirmed by
`wsl.exe -e printenv SSH_AUTH_SOCK` coming back empty even with the systemd
relay active and `SSH_AUTH_SOCK` correctly set in `/etc/environment`. Neither
`~/.zshrc`'s export nor `/etc/environment` ever reaches that git process, so
the systemd-relay fix (`docs/setup-sequence.md` item 15) alone doesn't fix
this case. The actual fix is `core.sshCommand` in `dot_gitconfig.tmpl`,
pointing at `~/.local/bin/git-ssh-with-agent`
(`dot_local/bin/executable_git-ssh-with-agent.tmpl`) — a wrapper that sets
`SSH_AUTH_SOCK` itself (reusing `.chezmoitemplates/ssh-agent-sock.sh.tmpl`)
before `exec ssh "$@"`, so git's ssh always has a working agent socket
regardless of how git itself was launched. Verify with the same
direct-`wsl.exe` invocation IntelliJ uses:
```bash
wsl.exe -d <distro> --cd <project-dir> -e git fetch origin --prune
```
If that still fails, confirm the wrapper is actually wired up:
```bash
git config --get core.sshCommand
ls -l ~/.local/bin/git-ssh-with-agent
```
and re-run `chezmoi apply -v` if either is missing/stale.

**IntelliJ fails a commit with `Couldn't get agent socket? / fatal: failed
to write commit object`, even though `fetch`/`push` work fine** — commit
signing (`gpg.format = ssh`) runs `ssh-keygen -Y sign`, a separate code path
from `core.sshCommand` that reads `SSH_AUTH_SOCK` straight from the process
environment. It hits the exact same `wsl.exe -e` bypasses-PAM gap as the
transport fix above, just at the signing step instead. Fixed the same way:
`gpg.ssh.program` in `dot_gitconfig.tmpl` points at
`~/.local/bin/git-ssh-sign-with-agent`
(`dot_local/bin/executable_git-ssh-sign-with-agent.tmpl`), which sets
`SSH_AUTH_SOCK` before `exec ssh-keygen "$@"`. Verify:
```bash
wsl.exe -d <distro> --cd <project-dir> -e git commit --allow-empty -m "test"
git config --get gpg.ssh.program
ls -l ~/.local/bin/git-ssh-sign-with-agent
```

**Docker image pulls were skipped** — Docker Desktop isn't installed/running,
or (WSL) its WSL2 integration isn't enabled. Install/start it, then re-run
`chezmoi apply`.

**ECR images specifically were skipped** — no AWS session was active when
`chezmoi apply` ran. Run `aws-login <profile>` first, then re-apply.

**`~/.aws/config` lost content you'd set up by hand** — chezmoi fully owns
and overwrites this file; anything not captured in `dot_aws/config.tmpl` is
wiped on apply. See `docs/setup-sequence.md` item 19, "AWS default region",
for how to fold existing content back in.

**A new alias/function/`PATH` entry "isn't there"** even after `chezmoi
apply -v` and `exec zsh` — two different causes, check in this order:
1. An already-running shell keeps the environment it started with; a plain
   `exec zsh` in that *same* shell should already cover this, but if you
   instead opened a brand-new terminal expecting it to pick up an edit made
   seconds ago, see #2.
2. The source dir (`~/.local/share/chezmoi`) is stale — `chezmoi apply`
   only renders whatever's already checked out there, it does not pull from
   GitHub first (see "Common commands" above). Check what commit it's
   actually on and whether that's really the one you expect:
   ```bash
   git -C ~/.local/share/chezmoi log --oneline -1
   ```
   then either `chezmoi update` (pulls + applies in one step) or, if
   `~/.local/share/chezmoi` is a symlink to a local working copy (see
   "Testing against a local, not-yet-pushed copy" below), edit that copy
   directly — there's nothing to pull.

**WSL disk usage keeps growing** — see "WSL2 disk space management" below.

For anything not covered here, [`docs/setup-sequence.md`](docs/setup-sequence.md)
has the full rationale for every step, including edge cases and the exact
error messages you might hit.

## Development / testing this repo

### Testing against a local, not-yet-pushed copy

Before this repo has a GitHub remote, or while iterating on a script, point
chezmoi at the working copy directly instead of a URL. From WSL, if the repo
lives on the Windows side (e.g. `C:\...\dotfiles`), it's reachable at
`/mnt/c/...`:

```bash
mkdir -p ~/.local/share
ln -s "/mnt/c/Andrew/Gravytrain/projects/dotfiles" ~/.local/share/chezmoi
chezmoi diff     # renders every template, shows what would change, runs no scripts
chezmoi apply -v # applies for real, prints each script's output
```

The symlink is what makes plain `chezmoi diff`/`chezmoi apply` work with no
extra flags afterwards — chezmoi always reads from `~/.local/share/chezmoi`
by default, and `chezmoi init --source=...` alone only affects that one
invocation, it isn't remembered.

Notes for iterating this way:
- `run_once_*` scripts only ever run once per machine (chezmoi tracks this in
  its own state, not in the repo). To force one to re-run after editing it,
  either change its content or wipe the tracked history:
  `chezmoi state delete-bucket --bucket=scriptState`.
- `run_onchange_*` scripts re-run automatically whenever their manifest
  changes, no extra step needed.
- Once you're happy with the result, push the repo to
  `gravytrain-willisa/dotfiles` on GitHub, then re-init from there into a
  path inside WSL's native filesystem (e.g. `~/dev/dotfiles`) for day-to-day
  use — see the note on `/mnt/c/` below.

### Testing a pushed branch before merging to main

Once a branch is pushed to GitHub, point `chezmoi init` at it directly with
`--branch` instead of waiting for it to merge to `main`:

```bash
chezmoi init --branch my-branch gravytrain-willisa/dotfiles
chezmoi diff     # renders every template, shows what would change, runs no scripts
chezmoi apply -v # applies for real, prints each script's output
```

Always inspect `chezmoi diff` before `chezmoi apply -v` here — branch testing
still targets your real home directory and runs real scripts, exactly like
testing against `main` would.

`--branch` only takes effect on that `init` — it isn't remembered for later
`chezmoi update` runs the way the source-dir symlink is for the local-copy
case above, so switching back once the branch merges means either `cd`-ing
into `~/.local/share/chezmoi` and `git checkout main` directly (the source
dir is just a normal git checkout under the hood), or re-running `chezmoi
init --branch main gravytrain-willisa/dotfiles` for a clean re-clone.

If chezmoi is already initialized against this repo on the machine (i.e.
`~/.local/share/chezmoi` already exists, either as a real clone from
first-time use or as the symlink from the local-copy case above), re-running
`chezmoi init --branch` won't switch it — `git checkout my-branch` in the
source dir directly instead.

## Repo layout

```
Shared config / data
  .chezmoidata/dotfiles.yaml              shared data/config (see "How to customize")
  .chezmoitemplates/brew-shellenv.sh.tmpl shared brew-PATH snippet
  .chezmoitemplates/ssh-agent-sock.sh.tmpl shared 1Password-agent/relay snippet
  .chezmoitemplates/sudo-timeout-backstop.sh.tmpl shared sudo-cleanup-backstop snippet

Package manifests (editable, additive-only — see "How to customize")
  dot_config/dotfiles/brew-packages.txt
  dot_config/dotfiles/sdkman-packages.txt
  dot_config/dotfiles/nvm-versions.txt
  dot_config/dotfiles/node-globals.txt
  dot_config/dotfiles/pyenv-versions.txt
  dot_config/dotfiles/pip-packages.txt
  dot_config/dotfiles/docker-images.txt
  dot_config/dotfiles/git-repos.txt
  dot_config/dotfiles/beanstalk-github-migrations.txt
  dot_config/dotfiles/op-env-vars.txt

Shell/profile templates -> target files
  dot_zshrc.tmpl                        -> ~/.zshrc
  dot_gitconfig.tmpl                    -> ~/.gitconfig
  dot_config/git/allowed_signers.tmpl   -> ~/.config/git/allowed_signers
  dot_aws/config.tmpl                   -> ~/.aws/config
  private_dot_ssh/config.tmpl           -> ~/.ssh/config (dir mode 0700)
  dot_config/systemd/user/ssh-agent-relay.service -> ~/.config/systemd/user/ssh-agent-relay.service (WSL only)

Utility scripts -> target files
  dot_local/bin/executable_connect-to-aws-jump-server.sh.tmpl
    -> ~/.local/bin/connect-to-aws-jump-server.sh
  dot_local/bin/executable_git-ssh-with-agent.tmpl
    -> ~/.local/bin/git-ssh-with-agent (used via dot_gitconfig.tmpl's core.sshCommand)

Install scripts, in execution order (see docs/setup-sequence.md)
  run_once_before_0000-configure-sudo-timeout.sh.tmpl  (runs first)
  run_once_before_0001-install-zsh.sh.tmpl
  run_once_before_0002-install-oh-my-zsh.sh.tmpl
  run_once_before_0003-install-homebrew.sh.tmpl
  run_onchange_0004-brew-packages.sh.tmpl
  run_once_0005-install-sdkman.sh.tmpl
  run_onchange_0006-sdkman-packages.sh.tmpl
  run_once_0007-install-nvm.sh.tmpl
  run_onchange_0008-nvm-versions.sh.tmpl
  run_onchange_0009-node-globals.sh.tmpl
  run_once_0010-install-pyenv.sh.tmpl
  run_onchange_0011-pyenv-versions.sh.tmpl
  run_onchange_0012-pip-packages.sh.tmpl
  run_onchange_0013-docker-images.sh.tmpl
  run_once_0014-install-1password-ssh-relay.sh.tmpl
  run_once_0015-install-aws-vault.sh.tmpl
  run_once_after_0016-handoff-shell-to-brew.sh.tmpl
  run_once_after_0017-handoff-chezmoi-to-brew.sh.tmpl
  run_onchange_0018-clone-git-repos.sh.tmpl
  run_once_0019-configure-powershell-profile.sh.tmpl
  run_onchange_0020-enable-ssh-agent-relay-service.sh.tmpl
  run_once_0021-install-junie-linux.sh.tmpl             (Linux only)
  run_once_after_0022-remove-sudo-timeout.sh.tmpl       (Linux only, runs last)
```

## WSL2 disk space management (Windows side, manual)

WSL2 stores each distro as a virtual disk (`ext4.vhdx`) that grows as you
write data but does **not** automatically shrink when files are deleted —
freed space becomes free space *inside* the VHDX, invisible to Windows, so
left unmanaged it just keeps expanding. None of this is chezmoi-managed —
it's all on the Windows side, outside anything WSL/macOS dotfiles can reach
— but it's easy to forget the specifics between machines, so it's
documented here rather than only in the cross-platform dev setup doc this
repo is built from.

**Enable sparse VHDX (Windows 11 22H2+, one-time per distro):**
```powershell
wsl --manage <DistroName> --set-sparse true
```
If your WSL version requires it, rerun with `--allow-unsafe` appended.
Sparse-VHDX support and flags vary by WSL version, so check `wsl --help` if
either form is rejected — older WSL builds don't have this command at all.
Reclaims space automatically as files are deleted going forward. It does
**not** retroactively shrink a disk that's already bloated — for that,
compact it manually:
```powershell
wsl --shutdown
diskpart
# inside diskpart:
select vdisk file="C:\Users\<you>\AppData\Local\Packages\<DistroPackage>\LocalState\ext4.vhdx"
compact vdisk
```
Find the exact path via `wsl --list --verbose` and the distro's package
folder, or:
```powershell
Get-ChildItem -Path "$env:LOCALAPPDATA\Packages" -Filter "ext4.vhdx" -Recurse
```

**Docker Desktop has its own, separate VHDX** (`docker-desktop-data`, if
using the WSL2 backend) that grows independently from images/containers/
build cache — manage it the same way (again, add `--allow-unsafe` only if
your WSL version requires it):
```powershell
wsl --manage docker-desktop-data --set-sparse true
```
Docker-level pruning is often the actual source of growth here, more so
than the VHDX setting itself:
```bash
docker system prune -a --volumes
```

**Cap growth up front via `.wslconfig`** (`%UserProfile%\.wslconfig`,
Windows side) — no direct "max VHDX size" key exists, but capping memory/
swap indirectly limits how much scratch space builds/layer caching can
consume:
```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
```

**Ongoing hygiene:**
- Repos already live in the Linux filesystem, not `/mnt/c/` (see "Notes"
  below) — so `node_modules`, Gradle/Maven caches, and build artifacts live
  inside the VHDX and are worth pruning periodically (`pnpm store prune`,
  `./gradlew clean`, `mvn clean`).
- SDKMAN/nvm caches (`~/.sdkman/candidates`, `~/.nvm/versions`) accumulate
  old JDK/Node versions — `sdk uninstall java <old-version>`, `nvm
  uninstall <old-version>`.
- `update-all` already runs `apt clean`/`brew cleanup` as part of its
  normal cycle.
- Always run `wsl --shutdown` before compacting — the VHDX can't be resized
  while the VM is running.

**Monthly maintenance, roughly:**
```bash
# Inside WSL, before shutting down
docker system prune -a --volumes -f
pnpm store prune
```
```powershell
# Then from Windows
wsl --shutdown
diskpart
# select vdisk file="..." ; compact vdisk
```
With sparse mode enabled, this becomes mostly self-maintaining rather than
a manual chore — worth still doing occasionally, since compaction isn't
automatic/instant even with sparse mode on.

## Notes

- This repo assumes real development happens inside WSL2 (Ubuntu) on
  Windows, not native Windows — see the cross-platform dev setup reference
  doc this was built from for the full rationale. That includes not living
  under `/mnt/c/...` long-term (slow `node_modules` installs, sluggish `git
  status`, Docker bind-mount weirdness) — the symlink trick above is a
  deliberate, temporary exception for testing before the repo has a home in
  WSL's native filesystem.
- Pin per-project tool versions with `.sdkmanrc`, `.nvmrc`, and
  `.python-version` inside each project repo — the manifests here are only
  for genuinely global, always-want-it tools. SDKMAN's `sdkman_auto_env` is
  enabled (see `docs/setup-sequence.md` item 6), so `cd`-ing into a directory
  with a `.sdkmanrc` automatically runs `sdk use` for you — no need to run it
  by hand.
- Per-directory git identity (`~/.gitconfig` `includeIf`) and `~/.ssh/config`
  per-host identity aliases, both covered in the cross-platform dev setup doc
  this repo is built from, are deliberately not managed here yet — this
  environment is currently single-purpose, so there's no separate
  identity/host to switch between, and doing it properly means deciding on
  clone-URL/host-alias conventions first rather than guessing at them.
  `~/.gitconfig` and `~/.ssh/config` themselves are chezmoi-managed (see
  [`docs/setup-sequence.md`](docs/setup-sequence.md) items 23 ("Git config")
  and 16 ("`~/.ssh/config`")) — it's specifically the per-context identity
  switching inside them that's deferred.

## Maintenance notes for agents

[CLAUDE.md](CLAUDE.md) (also linked from [AGENTS.md](AGENTS.md)) has the
full set of repo-editing conventions — naming scheme, manifest/script
pairing, template gotchas, and exactly what needs to stay in sync when you
change something. In short:

- Prefer editing a manifest under `dot_config/dotfiles/` or
  `.chezmoidata/dotfiles.yaml` over hardcoding a value in a script.
- Keep `dot_zshrc.tmpl`'s `help` output, this README's tables above, and
  `docs/setup-sequence.md`'s numbered walkthrough in sync whenever you add,
  remove, or rename a function, alias, manifest, or script.
- Manifests are additive-only by design — don't add uninstall behavior
  unless explicitly asked.
- Don't assume a WSL-exported env var is visible to a Windows-native binary.
- Don't script manual security/GUI choices (1Password settings, PowerShell
  execution policy, Windows Terminal profile) — these are genuinely manual,
  not automatable; see the checklist above for why.
