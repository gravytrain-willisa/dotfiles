# Detailed setup sequence

This is the full, ordered, step-by-step walkthrough of everything `chezmoi
apply` does — the canonical explanation of *why* each piece exists, not just
*what* it does. See [`README.md`](../README.md) for the quick-start, manual
setup checklist, common commands, and everyday usage; come here when you need
the rationale behind a specific step, or before changing one (see CLAUDE.md's
"Keep README.md in sync" rule — the same rule applies to this doc).

## What it sets up, in order

0. **Passwordless sudo for the bootstrap commands (Linux only)** — runs
   before everything else, so that the whole apply costs **one** password
   prompt instead of several.
   [`run_once_before_0000-configure-sudo-timeout.sh.tmpl`](../run_once_before_0000-configure-sudo-timeout.sh.tmpl)
   installs `/etc/sudoers.d/dotfiles-sudo-timeout`, granting the current user
   `NOPASSWD` for the exact commands the rest of this apply runs as
   sudo — `apt-get update`, `apt-get install -y *`, `tee -a /etc/shells`, and
   `chsh -s * <user>`. A first-ever apply needs sudo in six separate
   scripts — zsh (step 1), Homebrew's apt prerequisites (3), `unzip` for
   SDKMAN (7), pyenv's build deps (11), `socat` (15), and `chsh` at the
   handoff (20) — and only this first script's own `visudo`/`install` calls
   (installing the drop-in itself) still prompt normally, since the rule
   they install doesn't exist yet to cover them. It prints a one-line
   explanation before that prompt rather than letting a bare
   `[sudo: authenticate] Password:` be the first thing a fresh install
   shows.

   It's `before_0000`, with the rest of the install sequence starting at
   `before_0001-install-zsh`, so there's no alphabetical tie-break involved —
   every script has a distinct `NNNN`. This step used to be numbered 19, near
   the *end*, where it would have correctly widened the window for every
   *later* session but done nothing for the apply installing it.

   **Never edits `/etc/sudoers` directly** — always a real risk, since a
   syntax error there can lock sudo out entirely — it writes to a temp file,
   validates it with `sudo visudo -cf`, and only installs it if that passes.

   Scoped `NOPASSWD` rules, not a raised timestamp, because a cached-credential
   approach turned out not to be reliable enough here — worth recording so
   it isn't re-attempted:
   - Raising sudo's `timestamp_timeout` (Ubuntu's default is 15 minutes) via
     a per-user `Defaults:user timestamp_timeout=60` drop-in. This distro
     ships **sudo-rs** (the Rust reimplementation of sudo, now default on
     recent Ubuntu WSL images; recognizable by its nonstandard
     `[sudo: authenticate] Password:` prompt, versus classic sudo's
     `[sudo] password for <user>:`). Per-user `Defaults:user ...` entries
     parse cleanly under sudo-rs 0.2.13 (`visudo -cf` accepts them) but are
     silently ignored at runtime.
   - A global `Defaults timestamp_timeout=60` (no username) does take
     effect, but even 60 minutes isn't a hard guarantee on a first-ever
     apply — Homebrew's own bootstrap plus every formula in
     `brew-packages.txt`, then the pyenv Python builds later, can add up to
     longer than that.
   - A background loop refreshing the credential (`sudo -n -v` every 60s)
     so no later script re-authenticates mid-apply. Backgrounding with
     `disown` doesn't survive: chezmoi runs each script as its own process
     group and cleans up that whole group once the script exits, killing a
     plain `&`-backgrounded child regardless of `disown` (`disown` only
     stops bash itself from sending SIGHUP on exit — it does nothing
     against an external process-group kill).
   - Detaching that loop with `setsid` instead survives the process-group
     cleanup, but most likely then fails at the one thing it exists to
     do — sudo-rs's timestamp record is scoped to the terminal you
     authenticated in (`tty_tickets` and its replacement
     `timestamp_type=global`, either of which would share the cached
     credential more broadly, are rejected outright as unknown settings
     under sudo-rs 0.2.13), and `setsid` detaches from the controlling
     terminal entirely — so `sudo -n -v` run from inside that new session
     has no matching credential to refresh.

1. **Shell** — installs `zsh` (via apt on Linux; expected preinstalled on
   macOS) and sets it as the default login shell. The chosen shell name lives
   in [`.chezmoidata/dotfiles.yaml`](../.chezmoidata/dotfiles.yaml) as `shell:
   zsh`, so every script/template reads it from one place rather than
   hardcoding it — change it there if you ever switch shells.
2. **oh-my-zsh** — installed non-interactively, without touching
   `~/.zshrc` (chezmoi owns that file — see [`dot_zshrc.tmpl`](../dot_zshrc.tmpl)).
3. **Homebrew / Linuxbrew** — Homebrew on macOS, Linuxbrew on WSL/Linux
   (with the apt prerequisites it needs: `build-essential`, `procps`, `curl`,
   `file`, `git`).
4. **Brew packages** — installed from the plain-text list in
   [`dot_config/dotfiles/brew-packages.txt`](../dot_config/dotfiles/brew-packages.txt).
5. **Terminal prompt (starship)** — `starship` (installed via
   `brew-packages.txt`, item 4 above) replaces oh-my-zsh's own theme as the
   prompt, so both macOS and WSL show the exact same prompt from one shared
   binary instead of oh-my-zsh's theme system, which historically needed
   platform-specific font/glyph tweaks to render identically on both.
   `dot_zshrc.tmpl` sets `ZSH_THEME=""` (oh-my-zsh's own prompt disabled,
   rather than left to silently lose a "last one wins" race with starship)
   and initializes starship immediately after oh-my-zsh loads — starship has
   to go second, since `oh-my-zsh.sh` unconditionally sets `$PROMPT` itself
   on load and would otherwise clobber it. No custom `starship.toml` yet —
   it currently runs on starship's own default prompt/config, which is
   already consistent across both platforms; only worth adding once there's
   an actual desired customization to pin down, not preemptively.

   **Windows Terminal's default profile** is a separate, manual, Windows-side
   step this repo doesn't manage: Settings → Startup → Default profile → your
   WSL Ubuntu distro. Not automated because Windows Terminal keeps all
   profiles/themes/keybindings in one `settings.json` blob — a chezmoi
   template would either have to own that entire file (clobbering anything
   else you set there, e.g. other profiles or a colour scheme) or
   surgically patch one field inside someone else's JSON, which is more
   fragile than it's worth for a once-per-machine click. macOS has no
   equivalent step — iTerm2/Terminal.app just run the same zsh + dotfiles
   directly.

   **Native Windows PowerShell (WSL only)** — the starship binary above only
   covers prompts drawn *inside* WSL; PowerShell is a genuinely separate
   Windows process with its own PATH and its own profile, same
   "Windows-side binary" story as `aws-vault.exe`/`op.exe` (item 17 below).
   [`run_once_0020-configure-powershell-profile.sh.tmpl`](../run_once_0020-configure-powershell-profile.sh.tmpl)
   installs `starship.exe` via `winget` if missing, then configures **each**
   of Windows PowerShell (`powershell.exe`, 5.1, always present) and
   PowerShell 7+ (`pwsh.exe`, an optional separate install) that's actually
   found on the machine — the two have independent profiles. For each, it
   adds `Invoke-Expression (&starship init powershell)` to `$PROFILE` —
   asked of that PowerShell binary itself rather than assumed as a fixed
   path, since `$PROFILE` isn't guaranteed to be there already (PowerShell
   doesn't create it up front — a fresh machine genuinely has no file at
   that path until something writes one, not a broken link) and differs
   between `powershell.exe`/`pwsh.exe` (and can vary further by host).
   Append-only, not fully chezmoi-owned like `~/.zshrc` — PowerShell is
   fallback-only here (see top), so this only injects the one line it cares
   about via an idempotent `grep` check, leaving the rest of the profile
   (anything you add by hand) untouched.

   **One manual step this can't do for you:** Windows PowerShell's default
   execution policy (`Restricted`) blocks running *any* script file,
   including this profile — you'll see `... cannot be loaded because
   running scripts is disabled on this system` the first time PowerShell
   tries to load it. This is a genuine security control, not a config
   value, so the script only detects it and prints a warning (per binary
   found) rather than changing it for you. Fix once per PowerShell binary,
   no admin needed:
   ```powershell
   Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
   ```
   `RemoteSigned` is the standard dev-machine middle ground — locally
   created scripts (this profile, or anything else of yours) run freely,
   while anything downloaded from the internet still needs a valid
   signature.
6. **SDKMAN!** — installed via the official curl installer, skipped if a
   `sdkman-cli` brew formula is already present. **Deliberately leaves
   `sdkman_auto_answer` at SDKMAN's own default (`false`)** — an earlier
   version of this script set it to `true` to stop `sdk upgrade` blocking
   on prompts, but that setting auto-accepts *every* SDKMAN prompt,
   including "switch your local default to whatever upstream currently
   recommends." In practice that silently started downloading a different
   JDK vendor/version than the one pinned in `sdkman-packages.txt`, fighting
   the exact thing that manifest exists to enforce. `update-all` (item 26)
   avoids the prompt a different way instead: it just doesn't call `sdk
   upgrade` at all. If a previous apply already flipped this to `true` on a
   given machine, this script explicitly resets it back to `false`.
7. **SDKMAN candidates** — installed from
   [`dot_config/dotfiles/sdkman-packages.txt`](../dot_config/dotfiles/sdkman-packages.txt)
   (ships with Java 21 and Java 11 Corretto — 21 pinned as the default via
   the `default` marker described below — plus Gradle 9 and Maven 3).
   `dot_zshrc.tmpl` also exports `M2_HOME` pointed at SDKMAN's
   `candidates/maven/current` symlink, for tools/IDEs (e.g. IntelliJ) that
   look for that env var rather than resolving `mvn` off `PATH` — it stays
   correct automatically since that symlink always tracks whichever Maven
   version is currently active.
8. **nvm** — installed via the official install script, skipped if an `nvm`
   brew formula is already present. Which Node versions get installed is a
   separate, manifest-driven step (next).
9. **Node versions** — installed from
   [`dot_config/dotfiles/nvm-versions.txt`](../dot_config/dotfiles/nvm-versions.txt)
   (ships with Node 24 — pinned as the default via the `default` marker,
   same convention as `sdkman-packages.txt` — plus Node 12). `corepack
   enable` is run for every version installed, since its shims live under
   that version's own bin directory rather than being shared across
   versions, so pnpm/yarn versions come from each project's `package.json`
   `packageManager` field regardless of which Node version is active.
10. **Node global packages** — installed from
   [`dot_config/dotfiles/node-globals.txt`](../dot_config/dotfiles/node-globals.txt)
   against **every** Node version in `nvm-versions.txt`, not just the
   default — most global tools don't work on older pinned versions (e.g.
   Node 12), so each line can carry an optional version constraint: `package
   [constraint...]`, e.g. `npm-check-updates >=16` or `something-old <=12`.
   A constraint is an operator (`>=`, `<=`, `==`/`=`, `>`, `<`) directly
   followed by a Node **major** version number, no space (`>=16`, not `>=
   16`); multiple constraints on one line are ANDed together (`>=16 <20` =
   Node 16 through 19). No constraint means "install on every version,"
   which is a behavior change from before (non-default versions used to get
   nothing at all) — existing entries with no constraint now install
   everywhere, including legacy versions, so review them if that's not
   wanted. This is major-version-only integer comparison, not full semver —
   no `^16`/`~16.2` ranges. Keep the list short regardless — anything
   project-specific belongs in that project's `devDependencies` instead.
11. **pyenv** — installed via the official `pyenv.run` installer, skipped if
    a `pyenv` brew formula is already present, plus the `-dev` libraries
    pyenv needs to build CPython from source. Which Python versions get
    installed is a separate, manifest-driven step (next).
12. **Python versions** — installed from
    [`dot_config/dotfiles/pyenv-versions.txt`](../dot_config/dotfiles/pyenv-versions.txt)
    (ships with one version, pinned as the default via the `default`
    marker, same convention as `sdkman-packages.txt`/`nvm-versions.txt`).
13. **pip packages** — installed from
    [`dot_config/dotfiles/pip-packages.txt`](../dot_config/dotfiles/pip-packages.txt)
    against whichever version `pyenv-versions.txt` set as global.
14. **Docker images** — pulled from
    [`dot_config/dotfiles/docker-images.txt`](../dot_config/dotfiles/docker-images.txt).
    Docker Desktop itself isn't installed by this repo — it's a separate GUI
    install on both macOS and Windows — so this step skips quietly if the
    `docker` CLI or daemon isn't reachable yet. On a genuinely fresh
    machine, install Docker Desktop, enable WSL2 integration if on Windows,
    then run `chezmoi apply` again to trigger the pulls. Keep this list to
    cross-project images (a base MySQL/Playwright image, etc.) — anything
    project-specific already lives in that project's `docker-compose.yml`.
    Each line is `image[:tag] [platform]` — the platform is optional and
    only needed for images that only publish one architecture (e.g.
    `sftp:alpine linux/amd64`), which otherwise pull the wrong build (or
    fail outright) on a non-matching host; when present it's passed through
    as `docker pull --platform <platform> <image>`.

    Several entries are ECR images (`<account>.dkr.ecr.<region>.amazonaws
    .com/...`), which need a fresh login token before they'll pull. The
    script can't reuse the `ecr_auth` shell function from `dot_zshrc.tmpl`
    for this — that only exists in your interactive zsh session, not this
    script's own separate process — so it logs in directly: it scans the
    manifest for distinct ECR registry hostnames, and for each one runs `aws
    ecr get-login-password | docker login`, using the account/region
    embedded in that hostname rather than the current shell's
    `AWS_REGION`/`AWS_ACCOUNT_ID` (so it's correct even for an image from a
    different account/region than whatever profile is currently active). If
    there's no AWS session at all (`aws sts get-caller-identity` fails), it
    skips just the ECR-hosted images with a message telling you to run
    `aws-login <profile>` first and re-apply — non-ECR images (`mysql`,
    `httpd`, etc.) still pull normally either way.
15. **1Password SSH agent relay (WSL only)** — installs `socat` (apt) and
    `npiperelay.exe` (downloaded straight from its
    [GitHub releases](https://github.com/albertony/npiperelay/releases) into
    `~/.local/bin` — WSL can run a Windows `.exe` from anywhere in its own
    filesystem via interop, no Windows-visible path needed) via
    [`run_once_0014-install-1password-ssh-relay.sh.tmpl`](../run_once_0014-install-1password-ssh-relay.sh.tmpl).
    Scoped to genuine WSL2, not just any Linux — the guard checks
    `.chezmoi.kernel.osrelease` for `"microsoft"` (present only under WSL2)
    alongside `.chezmoi.os` — since on bare-metal Linux there's no Windows
    named pipe to relay from and this would just waste effort.

    Launching the relay is separate from installing it, and lives in
    [`.chezmoitemplates/ssh-agent-sock.sh.tmpl`](../.chezmoitemplates/ssh-agent-sock.sh.tmpl):
    it points `SSH_AUTH_SOCK` at `~/.1password/agent.sock` and, if no socket
    is listening there yet, starts `socat`/`npiperelay` to bridge 1Password's
    Windows named pipe into it (then waits briefly for the socket to appear,
    since `socat` backgrounds before creating it). Same WSL2 guard as above;
    on macOS it points straight at 1Password's own Unix socket, no relay
    needed; bare-metal Linux falls through with `SSH_AUTH_SOCK` untouched.

    It's a shared snippet rather than a block inside `dot_zshrc.tmpl`
    specifically because chezmoi runs `run_*` scripts with **bash,
    non-interactively** — `~/.zshrc` is never sourced, so a script can't
    assume an interactive shell has already started the relay. On a
    first-ever apply that's guaranteed to bite: this step only *installs*
    `socat` and `npiperelay`, and no zsh has existed since. Step 24 therefore
    includes the same snippet before it clones anything; when it was only in
    `dot_zshrc.tmpl`, every SSH clone failed with `Permission denied
    (publickey)`.

    **This only sets up the plumbing.** You still have to, once, by hand:
    1Password desktop app → Settings → Developer → "Use the SSH Agent". It's
    a GUI toggle with no CLI/file equivalent, so nothing here can do it for
    you. Verify afterwards with `ssh-add -l` (should list keys held by
    1Password, not local files) and `ssh -T git@github.com` — first
    connection to a new host auto-accepts its key rather than hanging on an
    interactive prompt (see "`~/.ssh/config`" below for why), without
    disabling the check entirely: it still fails loudly if that key ever
    unexpectedly changes later.
16. **`~/.ssh/config`** —
    [`private_dot_ssh/config.tmpl`](../private_dot_ssh/config.tmpl) →
    `~/.ssh/config` (the `private_` prefix keeps the `~/.ssh` directory
    itself at `0700`, as ssh expects). Deliberately minimal for now — a
    single global setting rather than per-host blocks:
    ```
    Host *
      StrictHostKeyChecking accept-new
    ```
    Previously set per-script (just for
    [`run_onchange_0018-clone-git-repos.sh.tmpl`](../run_onchange_0018-clone-git-repos.sh.tmpl),
    via a scoped `GIT_SSH_COMMAND`) so a first-ever connection to a new host
    (e.g. a Beanstalk/GitLab/self-hosted server never cloned from before)
    didn't hang a non-interactive script on "Are you sure you want to
    continue connecting (yes/no/[fingerprint])?". Centralising it here
    instead means it also covers plain interactive `ssh` and any future
    script, not just that one clone step — same `accept-new` behaviour as
    before (auto-trusts a host's key on first connection, then verifies
    against that stored key on every later connection, unlike
    `StrictHostKeyChecking=no` which disables the check permanently).

    No `IdentitiesOnly`/per-host `IdentityFile` pinning yet, and
    deliberately so: every identity currently lives only in the 1Password
    agent with no on-disk key file at all (per item 15's "genuinely
    vault-only" setup), and `IdentitiesOnly yes` only restricts ssh to
    identities it can find in on-disk `IdentityFile`s (default or
    explicit) — **not** "whichever key the agent happens to hold". Turning
    it on today, with nothing on disk to fall back to, would leave ssh with
    zero identities to try and break agent auth entirely. It only becomes
    genuinely useful once there's more than one key in the agent to
    disambiguate between, which is the same per-context-identity work as
    `~/.gitconfig`'s `includeIf` would be (item 23 below) — deferred for the
    same reason (see "Notes" in the README), not implemented as a guess
    here.
17. **aws-vault, backed by 1Password** — stores AWS credentials in
    1Password (the `op-desktop` backend) instead of plaintext in
    `~/.aws/credentials`, via
    [`run_once_0015-install-aws-vault.sh.tmpl`](../run_once_0015-install-aws-vault.sh.tmpl).
    This one genuinely installs a *different binary* per platform, because
    `op-desktop` authenticates the connecting process's identity
    (Authenticode signature on Windows) rather than just trusting whatever
    connects to a socket — the same relay trick used for SSH would connect
    as `npiperelay.exe`, which isn't a signed 1Password-aware binary, and
    would get rejected:
    - **macOS**: a normal `brew install aws-vault` — no VM boundary to
      cross, `op-desktop` talks to the local 1Password app directly. The
      1Password CLI also needs installing explicitly here (`brew install
      1password-cli`) — it's not bundled with the desktop app, and
      `op-desktop`'s "Integrate with 1Password CLI" check needs it present
      even though aws-vault never shells out to `op` itself.
    - **WSL**: installed as a genuine **Windows** binary, run from
      `run_once_0015-install-aws-vault.sh.tmpl` via `winget.exe` through WSL's
      interop:
      ```
      winget install --id ByteNess.AWSVault -e --silent --accept-package-agreements --accept-source-agreements
      ```
      so it runs as a real Windows process talking to the Windows-side
      1Password app. If `aws-vault.exe` isn't found in a new shell right
      after this installs, restart WSL (`wsl --shutdown` from Windows, then
      reopen) — WSL only re-reads the Windows `PATH` it inherits via
      interop at session start. `dot_zshrc.tmpl` adds an `aws-vault()`
      shell function so plain `aws-vault ...` on the command line
      transparently calls `aws-vault.exe` — you never need to type the
      extension.

      Being a real Windows process cuts both ways: `aws-vault exec
      <profile> -- aws ...` spawns that `aws` child command via a native
      Windows `CreateProcess` searching `%PATH%` — it can't see WSL's
      filesystem or the Linuxbrew-installed `aws` at all (`exec: "aws":
      executable file not found in %PATH%`, even though `which aws` inside
      WSL finds it fine). So the same script also installs the **Windows**
      build of the AWS CLI via `winget install --id Amazon.AWSCLI -e`,
      purely so aws-vault.exe's child process has something to find. This
      doesn't create a config conflict: `aws-vault` injects credentials as
      env vars into that child process directly rather than making it read
      `~/.aws/config`, so the fact that native `aws.exe` would otherwise
      look at a *separate* `%USERPROFILE%\.aws\config` (not WSL's
      `~/.aws/config`) never actually comes up in practice.

    **Important: env vars exported in `dot_zshrc.tmpl` do NOT reach
    `aws-vault.exe`.** WSL does not forward exported shell variables to
    invoked Windows processes by default — that requires explicitly listing
    the variable in `WSLENV` with the `/u` flag, which this repo doesn't set
    up. So anything `aws-vault`-related exported in `.zshrc` only affects
    tools run *from inside WSL* — it's silently invisible to `aws-vault.exe`
    itself, with no error to indicate it. Everything `op-desktop` needs has
    to be set as a **persistent Windows user environment variable**
    instead, via PowerShell's `[Environment]::SetEnvironmentVariable`
    (registry-backed, same effect as `setx` but without `setx`'s obscure
    1024-character value-length truncation bug). None of these have a
    config file backing them in this repo, so if a value ever needs to
    change, re-run the relevant command by hand. **Every one of these
    requires closing the terminal window entirely and opening a genuinely
    new one to take effect** — re-running a command in the same window
    keeps using that session's original environment snapshot, which is the
    single most common reason this looks broken when it isn't.

    **Manual one-time setup, on Windows, in the 1Password desktop app**
    (nothing here can script a GUI toggle):
    1. Settings → Security → turn on **"Unlock using Windows Hello"** —
       this is a hard prerequisite for the next step; without it the CLI
       integration toggle either stays greyed out or the prompts fail.
    2. Settings → Developer → turn on **"Integrate with 1Password CLI"**.
       This is the same integration surface the `op` CLI and aws-vault's
       `op-desktop` backend both authenticate through — there's no
       separate aws-vault-specific setting.
    3. `winget` itself must be present (ships by default on Windows 11).

    **Required environment variables** (`op-desktop` fails outright without
    any of these three — none of them fall back to some sensible default
    despite how optional they sound in aws-vault's own docs):

    - **`AWS_VAULT_BACKEND`** — without this, aws-vault silently uses
      Windows' own default backend (`wincred`, Windows Credential Manager)
      instead of `op-desktop`. No error, the credentials just don't end up
      in 1Password.
      ```powershell
      [Environment]::SetEnvironmentVariable("AWS_VAULT_BACKEND", "op-desktop", "User")
      ```
    - **`AWS_VAULT_OP_VAULT_ID`** — which 1Password vault to store items
      in. Without it: `Unable to create a 1Password ... Desktop Integration
      keyring: Environment variable unset or empty: "OP_VAULT_ID"`. Find a
      vault's UUID with the 1Password CLI (`op.exe`, installed by the same
      `run_once_0015` script via `winget install --id AgileBits.1Password.CLI
      -e`):
      ```powershell
      op.exe vault list
      [Environment]::SetEnvironmentVariable("AWS_VAULT_OP_VAULT_ID", "<uuid-from-vault-list>", "User")
      ```
    - **`AWS_VAULT_OP_DESKTOP_ACCOUNT_ID`** — which signed-in 1Password
      account to use. Without it (or with the wrong value):
      `onepassword.NewClient returned an error: ... Error { msg: Account
      not found }`. **The `USER ID` column from `op.exe account list`'s
      default table output is the wrong value** — that's the *user's* ID
      within the account, not the account's own ID. Use the JSON output
      instead, which exposes the actual account UUID field:
      ```powershell
      op.exe account list --format=json
      ```
      Find the `account_uuid` field (not `user_uuid`) and use that:
      ```powershell
      [Environment]::SetEnvironmentVariable("AWS_VAULT_OP_DESKTOP_ACCOUNT_ID", "<account_uuid-from-json>", "User")
      ```

    After setting all three (**new terminal window required**), the first
    `aws-vault` command that touches 1Password (e.g. `aws-vault add
    my-profile --backend=op-desktop`) pops a Windows Hello / 1Password
    authorization prompt — approve it once. **In practice this reprompts on
    every single subsequent `aws-vault` invocation, not just periodically**
    — `aws-vault.exe` is a one-shot CLI, and each separate process opens a
    fresh connection to 1Password's integration API, which requires its own
    fresh authorization regardless of how recently the last one happened.
    There's no 1Password setting to change this for third-party
    integrations like aws-vault; see "Reducing 1Password prompts" below for
    the actual workaround. If a profile was added before all three vars
    were set correctly (so it ended up in Windows Credential Manager
    instead of 1Password), move it across:
    ```bash
    aws-vault remove <profile> --backend=wincred
    aws-vault add <profile> --backend=op-desktop
    ```

    Usage is otherwise the standard `aws-vault` workflow: `aws-vault add
    <profile>` prompts for the access key/secret and stores them as a
    concealed-field item in 1Password (titled `aws-vault: <profile>`,
    tagged `aws-vault` by default — override with the optional
    `AWS_VAULT_OP_ITEM_TITLE_PREFIX`/`AWS_VAULT_OP_ITEM_TAG`, same
    persistent-Windows-env-var rules as above); `aws-vault exec <profile>
    -- aws s3 ls` runs a command with temporary credentials pulled from
    there.

    **Reducing 1Password prompts.** Calling `aws-vault exec <profile> --
    aws ...` once per AWS command means one 1Password prompt per command,
    since each is a fresh `aws-vault.exe` process. `dot_zshrc.tmpl` defines
    an `aws-login <profile>` shell function to avoid this: it authorizes
    **once** via `aws-vault export --format=export-env`, then `eval`s the
    resulting temporary credentials directly into your *current* shell —
    every `aws ...` call afterwards in that same terminal reuses them
    directly with no further aws-vault or 1Password involvement, for the
    12-hour session duration it requests. A plain script can't do this
    (env vars set inside a script's own subprocess vanish when it exits) —
    it has to be a shell function that runs in your shell's own process,
    which is exactly what this is:
    ```bash
    aws-login gravytrain
    aws s3 ls          # no prompt
    aws ec2 describe-instances   # still no prompt, same 12h session
    ```
    Works identically on macOS and WSL since it just calls whatever
    `aws-vault` resolves to on that platform (the plain binary, or the
    `aws-vault.exe` wrapper function above).

    `aws-login` also exports `AWS_ACCOUNT_ID`. aws-vault has no concept of
    storing an account ID itself — its keyring entry is just access
    key/secret/session token — so rather than duplicating the account ID
    somewhere it could drift out of sync with whichever credentials are
    actually active, `aws-login` derives it fresh each time via `aws sts
    get-caller-identity` immediately after exporting the credentials above.

    `aws-login` also calls `codeartifact_auth <profile>` and `ecr_auth` (both
    defined in `dot_zshrc.tmpl`), so one `aws-login <profile>` leaves you
    fully authenticated for `aws`, any CodeArtifact registry configured for
    that profile, and `docker pull`/`push` against ECR, all from the same
    1Password approval:
    - **`codeartifact_auth`** — looks up the calling profile in
      `.codeartifact` (`.chezmoidata/dotfiles.yaml`), which is keyed by
      aws-vault profile name, each mapped to a **list** of domain configs
      since one profile/account can have more than one CodeArtifact domain,
      e.g.:
      ```yaml
      codeartifact:
        gravytrain:
          - domain: gravytrain
            repository: Triton
            npm_scopes: [gravytrain]
        kgm:
          - domain: kgm
            repository: java
            env: KGM_CODEARTIFACT_AUTH_TOKEN
      ```
      For every domain listed under the matched profile it gets a
      CodeArtifact authorization token, caches it to
      `~/.cache/codeartifact-token-<profile>-<domain>` (`chmod 600`, one
      cache file per profile+domain pair so tokens for different
      profiles/domains don't clobber each other) and skips the API call on
      later logins within the token's 12h life. It exports the token as
      `env` if set, otherwise as `CODEARTIFACT_AUTH_TOKEN` — give every
      domain within the same profile a distinct `env` if more than one of
      them is npm-backed, otherwise their tokens clobber each other in the
      shared default var. If `npm_scopes` is set, it also configures npm for
      every listed scope against that domain's registry — one CodeArtifact
      repository can be the upstream for more than one npm scope, so the
      `:registry` line is written per scope while the `_authToken` line is
      written once per repository. Rather than `aws codeartifact login
      --tool npm`, which writes the literal token into `~/.npmrc` as
      plaintext, it writes a `${<env>}` placeholder — npm expands `${VAR}`
      references in `.npmrc` against the environment at request time, so the
      token itself never touches `.npmrc`. Domains that aren't npm-backed
      (e.g. `kgm`'s Maven/Gradle repository) just omit `npm_scopes` and rely
      on the exported `env` var instead. A profile with no entry in
      `.codeartifact` is skipped with a warning rather than failing
      `aws-login` outright.
    - **`ecr_auth`** — runs `aws ecr get-login-password | docker login` for
      `<account-id>.dkr.ecr.<region>.amazonaws.com`, skipping it if
      `~/.docker/config.json` already has a fresh (<11h old) entry for that
      registry.

    Both functions bail out with a clear error if run directly without
    `AWS_ACCOUNT_ID` set (i.e. without having run `aws-login` first) — they
    aren't useful standalone, since both need the account ID and shared AWS
    credentials `aws-login` sets up.
18. **Env vars from 1Password (`op-login`)** — a generic version of
    `aws-login` (item 17) for anything that just wants a plain API
    key/token in an env var, not a full backend like aws-vault: tools such
    as `claude` (Claude Code, `ANTHROPIC_API_KEY`) and `codex` (the OpenAI
    Codex CLI, `OPENAI_API_KEY`) read their credential straight from the
    environment instead of an interactive `login` flow. Rather than a
    one-off shell function per tool, `dot_zshrc.tmpl` defines a single
    generic `op-login` that reads
    [`dot_config/dotfiles/op-env-vars.txt`](../dot_config/dotfiles/op-env-vars.txt)
    — a plain-text `ENV_VAR_NAME op://vault/item/field` mapping, one per
    line, same `#`-comment/blank-line convention as every other manifest in
    this repo — and exports every var it lists. Adding a new tool's key
    later is a one-line edit to that file, not a new function.

    **One-time setup — create the 1Password items**, then add a line to
    `op-env-vars.txt` for each. It ships with two example entries
    (Anthropic + OpenAI) already written in but commented out — uncomment
    the ones you want once their 1Password items exist:
    1. Get an Anthropic API key from the
       [Anthropic Console](https://console.anthropic.com/settings/keys) →
       Create Key.
    2. Get an OpenAI API key from the
       [OpenAI Platform](https://platform.openai.com/api-keys) → Create new
       secret key.
    3. In the 1Password desktop app, add each as an **API Credentials** item
       in the `Private` vault (the credential goes in that item type's
       `credential` field), titled exactly `Anthropic API Key` /
       `OpenAI API Key` to match the shipped `op-env-vars.txt` entries —
       or use whatever title/vault you like and edit the `op://` reference
       in `op-env-vars.txt` to match instead.

    **Snyk (token + org ID)** — not shipped commented-out like the two
    above since it needs a second, non-standard field on the same item, but
    set up the same way:
    1. Get an API token from the
       [Snyk account settings page](https://app.snyk.io/account) → **General
       → Auth Token**.
    2. Get the org ID from **[app.snyk.io](https://app.snyk.io) → Settings →
       General → Org ID** for whichever org you want `snyk test`/`snyk
       monitor` to target by default. Get this right — a wrong-but-real org
       ID (e.g. one you're not a member of) fails with a
       `SNYK-OPENAPI-0004 Not found` 404 that reads like a permissions bug
       rather than a typo.
    3. In 1Password, add both to a single **API Credentials** item in the
       `Private` vault titled `Snyk API Key`: the token goes in that item
       type's built-in `credential` field; the org ID needs a **custom text
       field added to the same item** (name it e.g. `org id` — API
       Credentials items don't ship with an org-ID field by default).
    4. Add both lines to `op-env-vars.txt`:
       ```
       SNYK_TOKEN op://Private/Snyk API Key/credential
       SNYK_CFG_ORG op://Private/Snyk API Key/org id
       ```
       `SNYK_CFG_ORG` (not `SNYK_ORG`) is the Snyk CLI's own convention —
       any `snyk config` key can be set via an equivalent `SNYK_CFG_<KEY>`
       env var, so this is the env-var form of `snyk config set org=...`,
       not something specific to this repo.

    **WSL vs macOS.** Same signed-binary story as `aws-vault` above: a
    relayed `op` CLI (over the SSH-agent socat/npiperelay relay) would be
    rejected by 1Password desktop's Authenticode check, so on WSL
    `op-login` — like `aws-vault` — shells out to the winget-installed
    `op.exe` through WSL interop rather than a Linux `op` binary (no extra
    install needed; `op.exe` is already installed by
    `run_once_0015-install-aws-vault.sh.tmpl` for the `AWS_VAULT_OP_VAULT_ID`
    lookup in item 17 above). On macOS it uses the brew-installed `op` CLI
    directly. Both need the same one-time **Settings → Developer →
    "Integrate with 1Password CLI"** toggle as `aws-vault` (item 17 above)
    — it gates `op` itself, not just the SDK integration aws-vault uses.

    **Usage:**
    ```bash
    op-login
    claude     # ANTHROPIC_API_KEY already set, no login prompt
    codex      # OPENAI_API_KEY already set, no login prompt
    snyk test  # SNYK_TOKEN + SNYK_CFG_ORG already set, no --org flag needed
    ```
    Every mapped var is fetched via a single `op inject` call built from
    the whole manifest, rather than one `op read` per line — each
    `op(.exe)` invocation is a fresh process that can re-trigger
    1Password's unlock prompt (same limitation documented under "Reducing
    1Password prompts" in item 17 above), so N entries still only cost one
    prompt, not N.

    **Partial failures.** A single batched `op inject` fails outright if
    even one referenced item/field has since been renamed or deleted — with
    no fallback that would mean one bad manifest line breaking every
    credential, including ones that are still fine. So if the batched call
    fails, `op-login` automatically retries by resolving each manifest line
    individually via `op read`, exporting whatever does resolve and
    printing a warning per line that doesn't — at the cost of one 1Password
    prompt per entry instead of one shared prompt, since that's now N
    separate `op(.exe)` processes.

    **Forgetting to run it.** Neither `op-login` nor `aws-login` (item 17)
    auto-run at shell startup — both can trigger a 1Password unlock prompt,
    and `aws-login` additionally cascades into `codeartifact_auth`/
    `ecr_auth` (a `docker login`, `npm config` writes), which is too
    disruptive to fire on every new terminal; `aws-login` also has no sane
    default `<profile>` to auto-run with. Instead, `dot_zshrc.tmpl` defines
    `dotfiles-login-reminder`, run once automatically at the end of every
    new shell's startup: it checks whether the vars in
    `op-env-vars.txt` are already set and whether `AWS_ACCOUNT_ID` is set,
    and prints a one-line reminder for whichever is missing. It only reads
    state already in the environment/on disk — no 1Password or AWS calls of
    its own — so it costs nothing and never blocks shell startup. It also
    always prints a reminder to run `help` (item 22) to see the full list of
    dotfiles-provided commands.
19. **AWS default region** —
    [`dot_aws/config.tmpl`](../dot_aws/config.tmpl) → `~/.aws/config`, with a
    single `[default]` section setting `region` from `.chezmoidata/
    dotfiles.yaml`'s `aws.default_region` (currently `eu-west-2`). Note this
    only covers the `[default]` profile — named profiles (including
    aws-vault ones) don't inherit region from it. The fix that actually
    covers "any profile missing a region" — the one that fixes AWS SDK
    errors like `STS: GetSessionToken ... Missing Region` — is
    `dot_zshrc.tmpl` exporting `AWS_REGION`/`AWS_DEFAULT_REGION` from the
    same data value; the SDK falls back to those regardless of which
    profile is active. **This only covers tools run from inside WSL**
    (Linuxbrew's `aws`, etc.) — WSL does **not** forward exported shell
    variables to invoked Windows processes by default (that needs the
    variable explicitly listed in `WSLENV` with the `/u` flag, which isn't
    set up here), so this export is invisible to `aws-vault.exe` even when
    it's launched from a WSL shell via the `aws-vault()` function. The
    actual fix for anything Windows-native — `aws-vault.exe`, `aws.exe`, or
    any other AWS tooling, whether invoked through WSL interop or directly
    from a native PowerShell/Windows Terminal session — is a **persistent
    Windows user environment variable**, set once in PowerShell:
    ```powershell
    [Environment]::SetEnvironmentVariable("AWS_REGION", "eu-west-2", "User")
    [Environment]::SetEnvironmentVariable("AWS_DEFAULT_REGION", "eu-west-2", "User")
    ```
    This persists to the Windows user environment (registry-backed, same
    effect as `setx` but without `setx`'s 1024-character value-length
    trap) — it takes effect in *new* terminal sessions, not the one you run
    it in. Change the region in one place
    (`.chezmoidata/dotfiles.yaml`) for the WSL side; this PowerShell
    command has no config file to read from, so keep the value in sync by
    hand if it ever changes.

    **Heads up if `~/.aws/config` already has content** (e.g. a
    `[profile ...]` block with `role_arn`/`mfa_serial` you set up by hand
    or via `aws-vault add`): chezmoi fully owns and overwrites this file
    once it's managed, so anything not captured in
    `dot_aws/config.tmpl` will be wiped on the next apply. If that applies
    to you, either paste the existing file's content so it can be folded
    into the template, or run `chezmoi add ~/.aws/config` yourself first to
    pull the current state into the source repo before it gets templated
    over.
20. **Handoff: shell and chezmoi both move under Homebrew** — both `zsh`
    (step 1) and `chezmoi` (bootstrapped via curl, see "Quick start" in the
    README) are installed before Homebrew exists on a fresh machine, since
    each solves its own chicken-and-egg problem (a shell to `chsh` into;
    a tool to run this whole apply in the first place). Once step 3 has
    installed Homebrew/Linuxbrew, these two last steps install the brew
    formulas for both and re-point `chsh`/remove the curl-installed binary
    accordingly, so from then on `brew upgrade` keeps both current — no
    separate update path to remember. Both run last on the *same* `chezmoi
    apply`, no second apply required.
21. **`connect-to-aws-jump-server.sh`** —
    [`dot_local/bin/executable_connect-to-aws-jump-server.sh.tmpl`](../dot_local/bin/executable_connect-to-aws-jump-server.sh.tmpl)
    → `~/.local/bin/connect-to-aws-jump-server.sh`, opens an SSH tunnel
    (`127.0.0.1:3306` locally) through the Gravytrain jump box to a given
    system's RDS cluster for a given stage, for connecting a local MySQL
    client to a database that's otherwise only reachable from inside the
    VPC:
    ```bash
    connect-to-aws-jump-server.sh <triton|kgm> <dev|staging|prod> <jump-box-ip>
    ```
    The jump box is started on demand and has no fixed IP, so that's still a
    required argument. The RDS cluster **endpoint** is no longer hardcoded,
    though: each stage's cluster identifier
    (`<stage>-<system prefix>-<random suffix>`, e.g.
    `prod-gt-triton-core-database-coredatabasecluster-0ee0jjp507zm`) carries
    an opaque, cluster-generated suffix that would silently go stale if the
    cluster were ever recreated (snapshot restore, Terraform/CDK rebuild),
    so the script instead looks it up at connect time via `aws rds
    describe-db-clusters`, matching on the stable `<stage>-<system prefix>`
    prefix. The system -> prefix mapping lives in `rds.systems` in
    `.chezmoidata/dotfiles.yaml` (`triton: gt-triton-core-database`, `kgm:
    kgm-broker-centre-databas`), not in the script, so adding a third system
    later is a one-line data change rather than a script edit. The jump
    box's SSH login user is likewise data-driven (`rds.jump_user`), not
    hardcoded. Requires AWS
    credentials already in the shell — run `aws-login <profile>` first; the
    script checks for these and exits with a clear message rather than a raw
    AWS SDK error if they're missing.

    `~/.local/bin` (where this and other chezmoi-managed scripts live) is
    not on `PATH` by default in a non-login zsh shell — that's a
    `.profile`/POSIX-sh convention, and zsh doesn't source `.profile` —
    so `dot_zshrc.tmpl` adds `export PATH="$HOME/.local/bin:$PATH"`
    explicitly near the top. **Requires a new terminal (or `exec zsh`) after
    the first `chezmoi apply` that adds this** — an already-open shell keeps
    its original `PATH` from before the export existed.
22. **Aliases** — a small "Aliases" block at the bottom of `dot_zshrc.tmpl`:
    - `docker-update-images` — re-pulls every image currently present
      locally (by repository:tag), picking up upstream updates for tags like
      `:latest` that don't otherwise get refreshed automatically.
    - `docker-prune-all` — `docker system prune --volumes -f`, reclaims
      disk space from stopped containers/dangling images/unused volumes.
    - `gradle-refresh-dependencies` — `./gradlew --refresh-dependencies
      clean build`, skipping `test` always. This is a **function, not a
      plain alias**, because `checkstyle*`/`spotbugs*` tasks should only be
      excluded (`-x`) if the project actually applies those plugins —
      excluding a task that doesn't exist fails the entire build. So it
      first runs `./gradlew tasks --all -q`, greps for anything matching
      `^(checkstyle|spotbugs)[A-Za-z0-9]*` (covers every source set —
      `checkstyleMain`, `spotbugsIntegrationTest`, whatever exists — not
      just the two most common), and only passes `-x` for tasks that showed
      up in that list. On a project with neither plugin, it degrades to a
      plain `--refresh-dependencies clean build -x test`.
    - `gradle-show-dependency-updates` — `./gradlew --refresh-dependencies
      dependencyUpdates` (the `com.github.ben-manes.versions` plugin).
    - `npm-recreate-package-lock` — blows away `package-lock.json` and
      `node_modules/` and reinstalls from scratch; useful when the lockfile
      itself seems to be the source of a dependency problem.
    - `git-fetch-all`/`git-cleanup-all` — run `git fetch --all --tags
      --force --prune --prune-tags`/`git gc --aggressive` respectively in
      every repo directly under `~/projects` (where item 24's clone script
      puts them, one directory per repo, no nesting). Best-effort, not
      all-or-nothing — each repo runs in its own subshell, so a failure in
      one doesn't abort the rest or leave the calling shell's `cwd` changed.
      An empty or missing `~/projects` is a silent no-op rather than an
      error.
    - `kgm-connect-to-jump-server`/`triton-connect-to-jump-server` — thin
      wrappers around item 21's `connect-to-aws-jump-server.sh` that fix the
      `<system>` argument (`kgm`/`triton` respectively), taking
      `<stage> <jump-box-ip>` as their own two arguments. **Functions, not
      aliases** — a plain `alias x='cmd ${1} ${2}'` can't take arguments at
      all; `$1`/`$2` in an alias body resolve against the *invoking shell's*
      positional parameters (usually empty), not whatever follows the alias
      on the command line, so the original `alias
      kgm-connect-to-jump-server='connect-to-aws-jump-server.sh kgm ${1}
      ${2}'` would always call the script with two empty arguments.
    - Direct server SSH aliases — one `alias <name>='ssh ubuntu@<host>'` per
      entry in `servers` in `.chezmoidata/dotfiles.yaml` (a flat name ->
      hostname map), generated via a `{{ range }}` rather than hardcoded, so
      adding a server is a one-line data change. All reachable as `ubuntu`
      using whichever key the 1Password SSH agent offers (item 16) — no
      per-server user or identity file.
    - `help` — prints this whole list (everything above except `ll`, which
      is a plain `ls` convenience alias, not a dotfiles feature) plus the
      current `servers` names, so it's discoverable without reading
      `dot_zshrc.tmpl` directly. The server part is generated the same way
      the aliases themselves are, so it can't drift out of sync with
      `.chezmoidata/dotfiles.yaml`; the rest of the list needs updating by
      hand whenever a function/alias here changes (see CLAUDE.md).

23. **Git config** — [`dot_gitconfig.tmpl`](../dot_gitconfig.tmpl) →
    `~/.gitconfig`, currently just:
    ```ini
    [core]
      autocrlf = false
    ```
    `false`, not `true` — `autocrlf true`'s whole job is converting LF to
    CRLF on checkout and back on commit, which matters when Windows-native
    tools/editors are writing CRLF into files. Nothing in this setup does
    that any more (per the "do real dev inside WSL/macOS" premise this repo
    is built from); `true` here would just add a redundant, occasionally
    surprising conversion step on top of nothing that needs converting.
    Each repo's own `.gitattributes` (`* text=auto eol=lf`, already the
    convention in this repo — see [`.gitattributes`](../.gitattributes) here
    as an example) is the actual enforcement backstop, checked in and
    applied regardless of the cloning machine's global git config;
    `autocrlf false` here just stops git's own global behaviour from
    fighting that per-repo setting.

    `[user]` (`name`/`email`) is set from `.chezmoidata/dotfiles.yaml`'s
    `git.name`/`git.email` — one global identity for now. No `includeIf`
    per-context identity switching yet — see "Notes" in the README for why
    that's deferred rather than guessed at.

    **Commit signing (optional)** — set `git.signing_key` in
    `.chezmoidata/dotfiles.yaml` to your SSH key's *public* half (safe to
    commit — it's not a secret) and `dot_gitconfig.tmpl` adds
    `[gpg] format = ssh` + `[commit] gpgsign = true`, signing every commit
    with the same key already sitting in the 1Password agent — no separate
    signing key to generate or rotate. One-time setup:
    1. Get the public key from whichever 1Password SSH Key item you already
       use for git auth: `op item get "<item name>" --fields "public key"`
       (or open the item in the 1Password app and copy it).
    2. Paste it into `git.signing_key`, then `chezmoi apply`.
    3. In GitHub → Settings → SSH and GPG keys → **New SSH key**, paste the
       same public key again, but set **Key type: Signing Key** (GitHub
       treats auth and signing as separate key slots — the same key can
       fill both).

    Left blank by default; `dot_gitconfig.tmpl` only adds the signing
    config once `git.signing_key` is set, so leaving it empty is a no-op,
    not a broken state.

    Verify the key is actually reachable through the agent *before* the
    apply that turns this on — `gpgsign = true` applies to every repo on the
    machine, so if the agent can't produce that key, every commit fails
    rather than just going unsigned:
    ```bash
    ssh-add -l   # the listed fingerprint must match git.signing_key's
    ```

    Signing and verifying are separate halves, and the config above only
    does the first.
    [`dot_config/git/allowed_signers.tmpl`](../dot_config/git/allowed_signers.tmpl)
    → `~/.config/git/allowed_signers` supplies the second, pairing
    `git.email` with `git.signing_key` (both read from
    `.chezmoidata/dotfiles.yaml`, so there's no third copy to keep in sync)
    and scoping it with `namespaces="git"` so the key is only vouched for as
    a git signer. `dot_gitconfig.tmpl` points `gpg.ssh.allowedSignersFile`
    at it, gated on the same `git.signing_key` condition — when signing is
    off the template renders empty and chezmoi omits the file entirely
    rather than leaving a stray one behind.

    What that actually changes is **trust**, not whether signing happened:

    | | `%G?` | `git verify-commit` |
    |---|---|---|
    | key listed in allowed_signers | `G` (good) | passes |
    | key absent, or file not configured | `U` (untrusted) | exits non-zero |

    Worth knowing before trusting a spot-check: **`git log
    --show-signature` prints `Good signature` in both rows** — an untrusted
    signature only loses its principal name, it isn't labelled as untrusted.
    `git verify-commit` and `%G?` are the signals that actually
    discriminate:
    ```bash
    git log -1 --format=%G?      # G = good and trusted, U = good but untrusted
    ```

    Note the principal is matched **by key, not by email** — git reports
    whatever principal is paired with the signing key and never cross-checks
    it against the commit's author or committer, so a mismatched address
    verifies just as happily under the wrong name. Keeping it equal to
    `git.email` is what makes the reported identity meaningful rather than a
    correctness requirement.

    All of this is purely local — **GitHub's "Verified" badge is independent
    of it** and comes from step 3 above. Commits can verify cleanly here and
    still show as Unverified on GitHub until the key is added a second time
    as a Signing Key, which looks like a signing failure but isn't one.
24. **Clone git repos** —
    [`dot_config/dotfiles/git-repos.txt`](../dot_config/dotfiles/git-repos.txt)
    lists **full SSH clone URLs**, one per line (e.g.
    `git@github.com:Gravytrain-UK/gt-front-end-utils.git`) — not just
    GitHub's `<org>/<repo>` shorthand, so this supports any host (GitLab,
    Bitbucket, self-hosted) on the same list, not only GitHub;
    [`run_onchange_0018-clone-git-repos.sh.tmpl`](../run_onchange_0018-clone-git-repos.sh.tmpl)
    clones each one into its own directory under `~/projects/`, named from
    the URL's last path segment with any `.git` suffix stripped, e.g.
    `~/projects/gt-front-end-utils`. Same manifest pattern as the other
    `run_onchange_*` steps — add a line, `chezmoi apply`, only the new repo
    gets cloned. Skips a repo entirely if its destination directory already
    exists — re-cloning into an existing directory just fails outright, and
    this manifest is additive-only like the others (it doesn't pull/update
    existing checkouts, delete one if its line is removed, or otherwise
    touch a repo it's already cloned).

    Clones over **SSH**, not `gh repo clone` — this deliberately doesn't
    depend on `gh auth login` having happened, and wouldn't even apply to a
    non-GitHub host anyway. SSH auth goes through the 1Password SSH agent
    relay from "1Password as SSH vault" above, which is already required
    for any git push/pull regardless of host, so cloning works the moment
    that's set up.

    It can't assume that relay is *running*, though, so the script includes
    [`.chezmoitemplates/ssh-agent-sock.sh.tmpl`](../.chezmoitemplates/ssh-agent-sock.sh.tmpl)
    (see step 15) to start it and export `SSH_AUTH_SOCK` itself — chezmoi
    runs scripts with bash, non-interactively, so `~/.zshrc` hasn't run and
    on a first-ever apply no interactive shell has ever existed. Then a
    single `ssh-add -l` preflight, before any directory is created, either
    proves the agent answers and holds a key or aborts with one actionable
    message (pointing at the 1Password "Use the SSH Agent" toggle) rather
    than letting every repo in the list fail identically with `Permission
    denied (publickey)`. A failed script isn't recorded as having run, so
    fixing the cause and re-running `chezmoi apply` retries it — no
    `chezmoi state delete-bucket` needed.

    Past that preflight, an individual `git clone` failure is **non-fatal**
    (`|| echo "Failed to clone ... — skipping" >&2`, matching the other
    manifest-driven scripts): with the agent already proven working, one
    failure means that one repo (access not granted, renamed, deleted) and
    shouldn't abort the apply, and so every later step, on its way past.
    `git clone` cleans up its own partial directory on failure, so a skipped
    repo doesn't leave a stub that the "already cloned" check would later
    mistake for a real checkout.

    Cloning from a host never connected to before (a Beanstalk/GitLab/
    self-hosted server, say — not just GitHub) hits SSH's first-connection
    prompt: `The authenticity of host '...' can't be established ... Are
    you sure you want to continue connecting (yes/no/[fingerprint])?` —
    which blocks forever in a non-interactive script, since there's no
    stdin to answer it. This is handled globally now via
    `StrictHostKeyChecking accept-new` in item 16's `~/.ssh/config`
    (previously scoped to just this script via `GIT_SSH_COMMAND`, now
    covers plain interactive `ssh` too) — auto-trusts a host on first
    connection and adds it to `~/.ssh/known_hosts`, same as manually typing
    `yes` once, without disabling verification against that stored key on
    *later* connections, unlike `StrictHostKeyChecking=no`.

    `git-repos.txt` and the manifest of images/repos elsewhere in this repo
    (`docker-images.txt`, etc.) reflect real internal infrastructure — keep
    this dotfiles repo **private**.

    **Beanstalk -> GitHub migrations** —
    [`dot_config/dotfiles/beanstalk-github-migrations.txt`](../dot_config/dotfiles/beanstalk-github-migrations.txt)
    lists `<beanstalk clone URL> <github clone URL>` pairs, one per line,
    for repos mid-migration off Beanstalk. The same script, right after the
    clone loop above, handles each pair once its beanstalk URL has actually
    been cloned (the clone itself still comes from `git-repos.txt`, same as
    any other repo — this manifest never clones anything itself): renames
    the checkout's `main`/`master` branch to `<branch>-beanstalk`, renames
    `origin` to `beanstalk`, then adds a fresh `origin` pointing at the
    GitHub URL — so `origin` always means "where we push now" and the
    Beanstalk history stays reachable under the old remote name rather than
    being lost. Idempotent (skips a repo that already has a `beanstalk`
    remote) and skips a repo not yet cloned, rather than failing either way.
    Once a repo's migration is finished and GitHub is the sole source of
    truth, remove its line here — this manifest is for the transition only.
25. **`gh auth login`** —
    [`run_after_0019-gh-auth-login.sh.tmpl`](../run_after_0019-gh-auth-login.sh.tmpl)
    checks `gh auth status` and, if not already authenticated, runs `gh auth
    login`. This is a plain `run_after_` script (no `run_once_`/
    `run_onchange_`), so it re-checks — cheaply, `gh auth status` is
    near-instant — on **every** `chezmoi apply` rather than once: gh's auth
    state can change independently of anything chezmoi tracks (token
    expiry, `gh auth logout`, a revoked token on GitHub's side), so a
    one-time `run_once_` would never notice if it later needed
    re-authenticating.

    After confirming (or completing) auth, it also checks that whichever
    account `gh` is actually authenticated as matches `github.username` in
    `.chezmoidata/dotfiles.yaml` (via `gh api user --jq .login`) and warns
    on the terminal if not — this is the one thing `github.username`
    actually does; `gh auth login` itself is OAuth device-flow, so it
    doesn't take a username up front.

    **`gh auth login` genuinely can't be fully automated** — it's an
    interactive OAuth device-flow: it prints a one-time code and a URL,
    which you open in a browser and approve by hand. Running it from this
    script doesn't remove that step, it just means you hit it as part of
    `chezmoi apply` (which is already an interactive terminal command)
    instead of needing to remember to run `gh auth login` separately
    afterwards. It only gates `gh`-specific features (`gh pr`, `gh issue`,
    etc.) — not cloning, per item 24 above.

    **Exception: a `GH_TOKEN` mapping in `op-env-vars.txt`** (the same
    manifest item 18 covers) skips the prompt entirely. Before falling back
    to the interactive flow, the script looks for a `GH_TOKEN` line, and if
    found, resolves the 1Password CLI binary itself — `op.exe` on WSL (see
    item 15's aside on why WSL only has the winget-installed binary, not a
    native `op`), plain `op` on macOS — since this script is its own
    non-interactive process and never sources `dot_zshrc.tmpl`'s `op()`
    wrapper function. It then runs `gh auth login --with-token` with the
    resolved value. No `GH_TOKEN` line (the default), or a failed 1Password
    read, falls straight through to the interactive login unchanged — this
    is an opt-in shortcut, not a requirement.
26. **`update-all`** — a `dot_zshrc.tmpl` function that updates everything
    this setup depends on in one go:
    ```bash
    update-all
    ```
    Runs, in order: `chezmoi update` (pulls and applies the dotfiles repo,
    re-triggering any `run_onchange_*` scripts whose manifest changed
    upstream); `brew update && brew upgrade -y` followed by `brew cleanup`
    (Homebrew/Linuxbrew formulas, then removes old formula versions and
    clears brew's download cache — otherwise both just accumulate
    indefinitely); on Linux only (no apt on macOS) `sudo apt update && sudo
    apt upgrade -y` followed by `sudo apt autoremove -y && sudo apt clean`
    (same idea — drops packages no longer needed by anything installed, then
    clears `/var/cache/apt/archives`); npm globals, once per Node version in
    `nvm-versions.txt` (each has its own global `node_modules`) — runs
    `npm-check-updates` (`ncu -g --target minor -u`) to auto-apply anything
    that isn't a semver-major bump, then a plain `ncu -g` to print any
    remaining (necessarily major) updates for a human to apply by hand,
    since a major bump can change a CLI's own behavior; skips any Node
    version that doesn't have `npm-check-updates` installed (only installed
    on Node >=22 per `node-globals.txt`) rather than erroring; and finally
    `sdk selfupdate && sdk update` (SDKMAN's own updater, plus refreshing its
    candidate metadata).
    **Deliberately no `sdk upgrade`** — unlike brew/apt, SDKMAN candidate
    versions are pinned in `sdkman-packages.txt` (same `default`-marker
    convention as `nvm-versions.txt`/`pyenv-versions.txt`), not left to
    "whatever's newest." `sdk upgrade` doesn't just bump patch versions, it
    can also prompt to switch your local default to upstream's current
    recommendation, and auto-answering that prompt (rather than skipping
    the command) silently pulled in a completely different JDK
    vendor/version than the one pinned here — see item 6. Bump the version
    in `sdkman-packages.txt` instead, same as any other manifest in this
    repo. Ends with `exec zsh`, since `chezmoi update` can change
    `dot_zshrc.tmpl` itself and this shell's already-running zsh process
    won't pick that up otherwise. Deliberately a manually-invoked function,
    not a background cron/launchd job like the
    cross-platform dev setup doc's §4 suggests — `sudo apt upgrade` wants
    your password and `sdk upgrade` can prompt per-candidate, so unattended
    automation would either hang waiting for input or silently skip those
    prompts. Scheduling this (launchd on macOS, a shell-startup staleness
    check on WSL) is still open — see the doc's §4 if that's wanted later.
## Manifest conventions

Each numbered manifest (`*.txt`) is a plain newline-delimited list — blank
lines and `#`-comments are ignored. Its matching `run_onchange_*` script only
re-runs when that specific file's contents change (chezmoi hashes the
manifest into the script itself via a `# hash:` comment), so editing one list
and running `chezmoi apply` reinstalls just what changed.

**These manifests are additive only.** Removing a line stops it from being
(re)installed on future machines, but doesn't uninstall anything already
present — `chezmoi apply` never runs `sdk uninstall`/`nvm uninstall`/`pyenv
uninstall`/`brew uninstall`/`pip uninstall`/`docker rmi` on your behalf. If
you no longer want something, remove it from the list *and* uninstall/remove
it yourself with the relevant tool.

`sdkman-packages.txt` lines are `candidate version [default]` — most
candidates only ever need one line, but when a candidate has more than one
(e.g. Java 21 and Java 11 both installed), add the literal word `default` as
a third field on the line that should win, rather than relying on install
order. `nvm-versions.txt` and `pyenv-versions.txt` follow the same
convention, just with `version [default]` as the two fields (e.g.
`24 default` / `12` for Node, `3.14.7 default` for Python).
