# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [chezmoi](https://www.chezmoi.io/) source repo that provisions a dev environment on macOS and WSL2 Ubuntu (`gravytrain-willisa/dotfiles`). There is no application code, build, or test suite — "development" here means editing chezmoi-templated dotfiles and provisioning scripts, and the way to verify a change is to render/apply it with chezmoi, not to run a test command.

Real development is assumed to happen inside WSL2/macOS, never under `/mnt/c/...` long-term — see the "Notes" section of [README.md](README.md).

## Commands

No chezmoi/Go toolchain directly in a Windows checkout (verify with `which chezmoi` before assuming otherwise), but if WSL2 is available on the machine, chezmoi is normally installed there and can render/verify a template edit without applying it — side-effect-free, doesn't touch the user's real config:

```bash
wsl.exe -e bash -lc 'chezmoi --source=/mnt/c/path/to/dotfiles execute-template < some-file.tmpl'
```

Otherwise, changes are verified by the user re-running these on their machine:

```bash
chezmoi diff       # render every template, show what would change, run no scripts
chezmoi apply -v   # apply for real, print each script's output
```

To iterate on a not-yet-pushed copy instead of the GitHub remote, symlink the source dir (see "Testing against a local, not-yet-pushed copy" in [README.md](README.md)):

```bash
ln -s "/mnt/c/Andrew/Gravytrain/projects/dotfiles" ~/.local/share/chezmoi
```

Forcing a `run_once_*` script to re-run after editing it (chezmoi tracks "already ran" in its own state, not the repo):

```bash
chezmoi state delete-bucket --bucket=scriptState
```

`run_onchange_*` scripts need no such trick — they re-run automatically whenever the manifest file they're hashed against changes.

## Architecture

### Templating and shared data

`.chezmoidata/dotfiles.yaml` is the single source of config values (shell choice, git identity, GitHub username, AWS default region, per-profile CodeArtifact config, RDS system prefixes, direct-SSH server hostnames) read by every `*.tmpl` file via chezmoi's Go templating (sprig functions available). Never hardcode a value in a script that already exists in this YAML — add/read it from there instead so there's one place to change it.

**chezmoi's template execution errors on missing map keys** — it does not fall back to Go's default "invalid reflect value is falsy" behavior. `{{ if .someOptionalField }}` throws `map has no entry for key "someOptionalField"` if a YAML map in a `range` doesn't set that key on every item. Use sprig's `hasKey` for existence checks on optional per-item fields instead:

```gotemplate
{{ if hasKey . "env" }}{{ .env }}{{ else }}CODEARTIFACT_AUTH_TOKEN{{ end }}
```

`.chezmoitemplates/*.tmpl` holds snippets shared across multiple scripts via `{{ template "name.tmpl" . }}` (e.g. `brew-shellenv.sh.tmpl` puts brew on `PATH`) — add here, not by copy-pasting, when the same few lines are needed in more than one script.

### Naming convention drives execution order and semantics

chezmoi's filename prefixes are load-bearing, not decorative:
- `run_once_NNNN-*` — runs exactly once per machine ever (state tracked by chezmoi, not the repo).
- `run_onchange_NNNN-*` — re-runs whenever its own rendered content changes; scripts of this kind hash an editable manifest into a `# hash:` comment specifically so that editing the manifest (not the script) is what triggers a re-run.
- `run_once_after_NNNN-*` / `run_after_NNNN-*` — ordering-guaranteed to run last / on every apply respectively.
- The `NNNN` numeric prefix (4 digits, zero-padded, room for future insertions without renumbering everything) is execution order across *all* scripts, not scoped per-category, and is kept strictly unique — `run_once_before_0000-configure-sudo-timeout` runs first specifically so the sudo credential cache is widened before anything else needs sudo. Don't reintroduce a duplicate number to force a tie-break — check the "Repo layout" list in [README.md](README.md) (or the fuller step-by-step in [docs/setup-sequence.md](docs/setup-sequence.md)) before inserting a new script, and renumber everything after the insertion point so every script keeps a distinct `NNNN` that reflects real dependency order (e.g. anything Homebrew-dependent must sort after step 3).
- `dot_*` → `~/.*`, `private_dot_*` → sets the target directory mode to `0700` (used for `~/.ssh`), `executable_*` sets the executable bit.

### Scripts must stay bash 3.2-compatible

Every script here starts `#!/bin/bash` — an absolute path, deliberately. `#!/usr/bin/env bash` would break the bootstrap, since `run_once_before_0000-configure-sudo-timeout` and `run_once_before_0001-install-zsh` run before Homebrew exists.

On macOS that means **bash 3.2.57 (2007)**, the last GPLv2 release, which is what Apple still ships. `/bin/bash` is on the read-only signed system volume and cannot be replaced; `bash` in `brew-packages.txt` installs to `/opt/homebrew/bin/bash` and does **not** change what these scripts run under. So the lowest common denominator is 3.2, not whatever `bash --version` reports in your WSL shell.

Practically: no `declare -A`/`local -A` (associative arrays), `mapfile`/`readarray`, `${var^^}`/`${var,,}` case conversion, `&>>`, `;;&` in `case`, `coproc`, `globstar`/`**`, `wait -n`, `${var@Q}`, or negative array indices. All bash 4+, all silently fine on WSL and broken on macOS — in the code path that is hardest to test (the macOS branches of `run_once_0003/0005/0010/0015`, `run_once_after_0016`, and the darwin `SSH_AUTH_SOCK` block have never been rendered or run). `bash -n` on a Linux box will **not** catch these; it parses them happily. Prefer POSIX-ish constructs — `while IFS= read -r` loops over a manifest, `case` for matching, `tr` for case conversion — which is what the existing scripts already do.

### Manifest + run_onchange script pairing

Most provisioning is a plain-text manifest under `dot_config/dotfiles/*.txt` (newline-delimited, `#`-comments and blank lines ignored) paired with one `run_onchange_*.sh.tmpl` that installs from it. Examples: `brew-packages.txt`, `sdkman-packages.txt`, `nvm-versions.txt`, `node-globals.txt`, `pyenv-versions.txt`, `pip-packages.txt`, `docker-images.txt`, `git-repos.txt`, `op-env-vars.txt`, `beanstalk-github-migrations.txt`.

**These manifests are additive-only.** Removing a line stops future installs but never uninstalls anything already present — no script here ever runs `*-uninstall`/`brew uninstall`/`docker rmi` on the user's behalf.

Several manifests share a convention worth knowing before editing them:
- `sdkman-packages.txt` / `nvm-versions.txt` / `pyenv-versions.txt`: `<value> [default]` — an optional literal `default` marker picks which installed version wins when more than one is present, rather than relying on install order.
- `node-globals.txt`: `package [constraint...]`, where a constraint is `>=`/`<=`/`==`/`=`/`>`/`<` immediately followed by a Node **major** version number (no space, no semver ranges) — multiple constraints on one line AND together. No constraint means "install on every Node version in `nvm-versions.txt`."
- `docker-images.txt`: `image[:tag] [platform]` — `platform` is optional, only for single-arch images.
- `beanstalk-github-migrations.txt`: `<beanstalk clone URL> <github clone URL>` — the beanstalk URL must also have its own active (uncommented) line in `git-repos.txt`, since this manifest only migrates a repo already cloned from there, never clones one itself. Remove a repo's line once its migration is complete.

### The AWS credential/auth chain (`dot_zshrc.tmpl`)

`aws-login <profile>` is the entry point for everything AWS-related in a shell session, and cascades:

```
aws-login <profile>
  → aws-vault export --format=export-env, eval'd into the current shell (12h session)
  → derives AWS_ACCOUNT_ID via aws sts get-caller-identity
  → codeartifact_auth <profile>   (looks up .codeartifact[<profile>] — a LIST of domain configs)
  → ecr_auth                       (aws ecr get-login-password | docker login)
```

`.codeartifact` in `.chezmoidata/dotfiles.yaml` is keyed by aws-vault profile name, each mapped to a list of `{domain, repository, npm_scopes?, env?}` entries — a profile can have more than one CodeArtifact domain. `env` defaults to `CODEARTIFACT_AUTH_TOKEN`; give every npm-backed domain within the same profile a distinct `env` or their tokens clobber each other. `npm_scopes` is only set for npm-backed domains — when present, `codeartifact_auth` writes a `${<env>}`-placeholder (not the literal token) into npm's registry config per scope, so the real secret only ever lives in the exported env var / the per-profile-per-domain cache file at `~/.cache/codeartifact-token-<profile>-<domain>`, never in `.npmrc` on disk.

Both `codeartifact_auth` and `ecr_auth` bail with a clear error if called directly without `AWS_ACCOUNT_ID` set — they assume `aws-login` ran first and are not meant to be called standalone.

**WSL-specific gotcha that recurs across this file**: shell-exported env vars in `dot_zshrc.tmpl` do **not** reach Windows-native binaries (`aws-vault.exe`, `op.exe`) launched from WSL — WSL only forwards vars explicitly listed in `WSLENV`, which isn't set up here. Anything those binaries need (`AWS_VAULT_BACKEND`, `AWS_VAULT_OP_VAULT_ID`, `AWS_REGION`, etc.) has to be a **persistent Windows user environment variable** set via PowerShell's `[Environment]::SetEnvironmentVariable(..., "User")`, and only takes effect in a brand-new terminal window. Keep this in mind before "fixing" a Windows-side env var problem by adding an export to `dot_zshrc.tmpl` — it will silently do nothing for the Windows-native binary.

### Everything else in `dot_zshrc.tmpl`

Also defines: `op-login` (generic 1Password → env var injection, driven by `op-env-vars.txt`), `dotfiles-login-reminder` (auto-runs on shell startup, read-only — checks whether `op-login`/`aws-login` have been run and reminds if not, no API calls of its own, and points at `help`), `update-all` (chains `chezmoi update` + brew + apt + SDKMAN updates), and the "Aliases" block at the bottom (`gradle-refresh-dependencies`, the `*-connect-to-jump-server` wrappers, and `git-fetch-all`/`git-cleanup-all` are shell functions rather than plain aliases specifically because they need positional arguments or looping — a plain `alias` can't do either). `git-fetch-all`/`git-cleanup-all` loop every repo under `~/projects` via `_git_all_projects`, a private helper (leading `_`, not listed in `help`) shared by both.

**Keep `help` in sync.** `help` (end of the Aliases block) prints a static list of every function/alias defined earlier in the file, plus a `{{ range .servers }}`-generated list of the direct-server SSH aliases. The server part regenerates itself automatically from `.chezmoidata/dotfiles.yaml`, but the static part does not — whenever a function or alias is added, removed, or renamed in `dot_zshrc.tmpl`, update `help`'s static list in the same change, or it silently drifts out of sync the same way README.md's alias list already had (see below).

### Keep README.md and docs/setup-sequence.md in sync

[docs/setup-sequence.md](docs/setup-sequence.md) is a numbered, step-by-step walkthrough of everything `chezmoi apply` does, in execution order, and is treated as the canonical explanation of *why* each piece exists — not just *what* it does. When changing the shape of a manifest, `.chezmoidata/dotfiles.yaml`, or a script's behavior, update the corresponding numbered item there (and the "Repo layout" file listing in [README.md](README.md)) in the same change, rather than letting it drift.

[README.md](README.md) itself is the day-to-day quick-reference — quick start, manual setup checklist, common commands, "how to customize" table, and troubleshooting. It links out to `docs/setup-sequence.md` for rationale rather than repeating it. When adding/removing/renaming a command, manifest, or manual step, update whichever of README's tables covers it (Common commands / How to customize / Manual setup checklist) in the same change, in addition to `docs/setup-sequence.md` and `help`'s static list (see above).
