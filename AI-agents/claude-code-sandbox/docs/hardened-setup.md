# Claude Code on Artix — Hardened Sandbox Setup

Tailored to the same case as the OpenCode guide: **your own local projects**,
CLI as the daily driver, no separate API billing (uses your Claude Pro/Max
subscription login instead of an API key), opsec-first.

Claude Code is a different animal from most agents in one important way:
**it ships its own OS-level sandbox**, built on bubblewrap on Linux — the
same tool this guide already uses for the outer wrapper. Anthropic's own
docs describe permissions and sandboxing as "complementary security
layers" meant to be used together, not alternatives. So this setup has
**two** bubblewrap layers plus OpenSnitch, rather than one:

1. **Claude Code's built-in sandbox** (`sandbox.enabled` in `settings.json`) — confines the Bash tool and everything it spawns, with OS-level filesystem and network enforcement, independent of what wraps the process.
2. **An outer bubblewrap wrapper** (`cclaude`, this repo) — confines the `claude` process itself from the moment it starts: fake home, cleared environment, one writable project directory. This is what stops the *rest* of Claude Code (Read/Edit/Write tools, MCP, its own config) from touching anything outside the project, since the built-in sandbox only governs the Bash tool.
3. **OpenSnitch** — network egress control underneath both, catching anything that slips past layers 1 and 2.

---

## Step 0 — Security model and its limits

What this gives you:

* Filesystem blast radius: one project directory, enforced twice (outer wrapper + built-in sandbox)
* Environment blast radius: five harmless variables plus `DISABLE_AUTOUPDATER` — zero secrets injected
* Credential blast radius: Claude Code's login token confined to a dedicated, wrapper-only state directory
* Network blast radius: only explicitly approved domains, enforced by both the built-in sandbox's proxy and OpenSnitch
* Config: `settings.json` is read-only inside the wrapper, so neither a compromised session nor a rogue project can loosen its own policy
* Permissions: `defaultMode: "default"` — every tool use asks first; `bypassPermissions` is disabled outright at the config level, not just avoided by habit

Honest limits:

* The agent can read everything in the mounted project directory.
* The built-in sandbox's default **read** policy is permissive by design (the whole computer is readable except denied paths) — this config explicitly denies `~/.ssh`, `~/.aws`, `~/.gnupg`, but that only matters if you ever run bare `claude` outside the wrapper. Inside the wrapper, those paths don't exist at all.
* The built-in sandbox's network proxy does not terminate/inspect TLS by default, so it filters by hostname, not content.
* This setup does not make the agent smart or honest — review permission prompts, especially anything touching credentials or the network.

---

## Step 1 — Install base packages

```bash
sudo pacman -S --needed bubblewrap socat opensnitch git
```

Explained:

* `bubblewrap` — the sandboxing primitive both layers use (our outer wrapper, and Claude Code's own built-in sandbox on Linux).
* `socat` — the relay Claude Code's built-in sandbox uses to route Bash-tool network traffic through its filtering proxy. Without it, the built-in sandbox can't enforce network restrictions.
* `opensnitch` — the outer, independent network firewall (Step 8).
* `git` — needed to build AUR packages; not required for your own project workflow.

If you don't have an AUR helper yet:

```bash
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin
less PKGBUILD
makepkg -si
cd .. && rm -rf paru-bin
```

---

## Step 2 — Install Claude Code

Anthropic does not publish a pacman/Arch repository (only apt, dnf, and apk), so on Artix the choice is between the AUR and the official native installer. **Prefer the AUR** — it keeps the binary owned and updated by your package manager, consistent with the rest of this setup, and community AUR packages disable Claude Code's own background auto-updater automatically (updates flow through `paru -Syu` instead).

```bash
paru -S claude-code-bin
```

(`claude-code-stable-bin` is the alternative if you want the ~1-week-delayed, regression-filtered channel instead of the latest release.)

When paru shows the PKGBUILD, verify:

1. `source=` points to `downloads.claude.ai/claude-code-releases/...` (the official release bucket)
2. checksums are present, not `SKIP`
3. no unexpected commands in `prepare()`/`package()`

Confirm it runs:

```bash
claude --version
```

Note: Arch/Artix is not an officially supported OS for Claude Code (officially: Ubuntu 20.04+, Debian 10+, Alpine 3.19+), but the binary itself is a standard Linux build and runs without issues in practice.

Avoid the native installer (`curl -fsSL https://claude.ai/install.sh | bash`) for this setup specifically — not because it's unsafe, but because it self-updates in the background by design, which conflicts with "pacman owns the binary." If you ever do use it, set `DISABLE_AUTOUPDATER=1` (already done for you inside the wrapper, Step 4) and update manually.

### Troubleshooting: "claude command at ~/.local/bin/claude missing or broken · run claude install to repair"

This warning comes from Claude Code's built-in updater looking for the binary at `~/.local/bin/claude`, which is the *native-installer* location. On an AUR install the binary lives elsewhere (`/usr/bin/claude`, provided by the package), so this check fails and prints the warning.

**Do NOT run `claude install` in response.** That would download and install a *second*, native copy of Claude Code alongside the AUR one, producing exactly the mixed-installation breakage the warning hints at. On an AUR-managed system it is the wrong repair.

Instead, confirm the AUR install is actually present and located correctly (run these in a normal terminal, outside the sandbox):

```bash
which -a claude
pacman -Q claude-code-bin claude-code-stable-bin 2>/dev/null
ls -la /usr/bin/claude
```

* If `which claude` shows `/usr/bin/claude` and the package is listed — Claude Code is installed correctly and the warning is harmless updater noise you can ignore. `DISABLE_AUTOUPDATER=1` in the wrapper (Step 4) suppresses the underlying update behavior anyway; the message is cosmetic.
* If `which claude` finds nothing and no package is listed — the install didn't complete. Re-run `paru -S claude-code-bin`, review the PKGBUILD, let it build, then verify with `claude --version`.

Because the wrapper's `PATH` is set to `/usr/local/bin:/usr/bin:/bin` (Step 4), a binary at `/usr/bin/claude` is reachable inside the sandbox. If your AUR package installs to a non-standard location instead (some variants use `/usr/lib/claude-code/` with a `/usr/bin` symlink), confirm the symlink exists — `ls -la /usr/bin/claude` should resolve to it.

---

## Step 3 — Hardened settings.json

Claude Code's config lives at `~/.claude/settings.json` (user-level). Because our outer wrapper replaces `$HOME` entirely, this file needs to live inside the **dedicated state directory** that gets bound into the sandbox — not your real home.

```bash
mkdir -p ~/.local/state/claude-code-sandbox
cat > ~/.local/state/claude-code-sandbox/settings.json << 'EOF'
{
  "$schema": "https://json-schema.org/claude-code-settings.json",
  "permissions": {
    "defaultMode": "default",
    "disableBypassPermissionsMode": "disable",
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.gnupg/**)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Bash(sudo:*)",
      "Bash(su:*)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": false,
    "allowUnsandboxedCommands": false,
    "filesystem": {
      "denyRead": ["~/.ssh", "~/.aws", "~/.gnupg"]
    },
    "network": {
      "allowedDomains": []
    }
  }
}
EOF
```

Explained:

* `"defaultMode": "default"` — every tool use prompts for confirmation. No auto-accept.
* `"disableBypassPermissionsMode": "disable"` — locks out `--dangerously-skip-permissions` and `defaultMode: "bypassPermissions"` at the config level. Even a future you, or a script, or a compromised session, cannot flip this on without editing the file — which, inside the wrapper, is read-only (Step 4).
* `"deny"` — explicit deny rules for SSH/AWS/GPG directories and `.env` files, plus `sudo`/`su`. These are **belt-and-suspenders**: the wrapper already hides these paths entirely, so this list matters mainly if you ever run `claude` unsandboxed by mistake.
* `"sandbox.enabled": true` — turns on Claude Code's own built-in bubblewrap-based sandbox for the Bash tool.
* `"failIfUnavailable": true` — if bubblewrap/socat aren't available for any reason, Claude Code refuses to start rather than silently falling back to running Bash commands unsandboxed. This is the setting Anthropic recommends for anyone who wants sandboxing to be a hard requirement, not a best-effort.
* `"autoAllowBashIfSandboxed": false` — kept at `false` here so **every** Bash command still prompts, matching a review-everything posture. Anthropic's own data says auto-allow cuts prompts by ~84% while keeping the same OS-level walls — if prompt fatigue becomes a problem, flipping this to `true` is a reasonable, documented trade: the boundary enforcement doesn't change, only whether you're asked first.
* `"allowUnsandboxedCommands": false` — if a command fails inside the sandbox, Claude Code cannot retry it unsandboxed. No escape hatch.
* `"filesystem.denyRead"` — reinforces the permission-layer deny rules at the sandbox-enforcement layer specifically.
* `"network.allowedDomains": []` — empty on purpose. Any Bash-tool network request falls through to the normal ask/deny permission flow instead of silently succeeding. You'll add domains here (or approve them interactively) as you actually need them.

This file is intentionally **not** committed to the state directory's git history if you ever put that under version control — treat it as machine config, not project config.

---

## Step 4 — The outer wrapper

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/cclaude << 'EOF'
#!/bin/sh
# cclaude — run Claude Code sandboxed in the current directory or a given project directory
# usage:
#   cclaude
#   cclaude /path/to/project
#   cclaude /path/to/project /login

set -eu

if [ -d "${1:-}" ]; then
  PROJECT="$(realpath "$1")"
  shift
else
  PROJECT="$(realpath .)"
fi

STATE="$HOME/.local/state/claude-code-sandbox"

exec bwrap \
  --ro-bind /usr /usr \
  --symlink usr/lib /lib \
  --symlink usr/lib /lib64 \
  --symlink usr/bin /bin \
  --symlink usr/bin /sbin \
  --ro-bind /etc/resolv.conf /etc/resolv.conf \
  --ro-bind /etc/ssl /etc/ssl \
  --ro-bind /etc/ca-certificates /etc/ca-certificates \
  --proc /proc \
  --dev /dev \
  --tmpfs /tmp \
  --dir /home \
  --tmpfs "$HOME" \
  --dir "$HOME/.claude" \
  --bind "$STATE" "$HOME/.claude" \
  --ro-bind "$STATE/settings.json" "$HOME/.claude/settings.json" \
  --bind "$STATE/legacy-claude.json" "$HOME/.claude.json" \
  --bind "$PROJECT" "$HOME/project" \
  --unshare-all \
  --share-net \
  --die-with-parent \
  --new-session \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv USER "${USER:-user}" \
  --setenv PATH "/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  --setenv DISABLE_AUTOUPDATER "1" \
  --chdir "$HOME/project" \
  claude "$@"
EOF
chmod +x ~/.local/bin/cclaude

# The wrapper binds a state directory and a legacy-file placeholder that
# must exist before first run (bwrap fails on a missing bind source):
mkdir -p ~/.local/state/claude-code-sandbox
touch ~/.local/state/claude-code-sandbox/legacy-claude.json
```

Check `~/.local/bin` is in your PATH (add to `~/.bashrc` if not, as in the OpenCode guide).

---

## Step 5 — Wrapper explained, line by line

Most of this mirrors the OpenCode wrapper exactly — same read-only `/usr`, same minimal `/etc`, same fake home, same cleared environment. The differences worth calling out:

### The `.claude` state directory, layered like `opencode.json` was

```sh
--dir "$HOME/.claude"
--bind "$STATE" "$HOME/.claude"
--ro-bind "$STATE/settings.json" "$HOME/.claude/settings.json"
```

Same pattern as the OpenCode config bind: the directory is read-write (so Claude Code can write its credentials file, session transcripts, shell snapshots, backups, and task state), but `settings.json` is re-mounted read-only on top of it. Net effect: Claude Code can persist everything it needs to function, but cannot modify its own policy file — sandboxing, permission mode, and deny rules stay locked regardless of what happens during a session.

Everything that lands in `~/.claude/` inside the sandbox persists on the host at:

```text
~/.local/state/claude-code-sandbox/
```

including `.credentials.json` after you log in (Step 6). Wipe it anytime to log out and reset all state:

```bash
rm -rf ~/.local/state/claude-code-sandbox/*
```

(You'll need to re-create/copy `settings.json` back afterward — see Step 3 — and `touch` a fresh `legacy-claude.json` in the state dir, since `rm -rf` removes both.)

### The legacy `~/.claude.json` file — persisted, with a caveat

```sh
--bind "$STATE/legacy-claude.json" "$HOME/.claude.json"
```

Claude Code writes a legacy file directly at `~/.claude.json` (note: not inside `.claude/`) holding onboarding state (your text style / theme choice), the per-directory trust list, recent project paths, and MCP server config. This wrapper binds it to a file in the state directory so it **persists across sessions**.

Without this bind, the file would live only in the sandbox's `tmpfs` fake home and vanish when the session ends — meaning every session re-runs onboarding (re-asks the text-style prompt), re-shows the "do you trust this folder?" dialog, and forgets project history. Persisting it fixes all three. Note that `.credentials.json` (your actual auth token) lives in `~/.claude/` and is already persisted by the directory bind above — so persisting the legacy file is specifically about onboarding/trust/history, not login itself.

**Privacy trade-off:** `~/.claude.json` accumulates an on-disk record of every project path you've opened. For your own local projects that's a reasonable trade for not re-onboarding each time. If you'd rather keep zero persistent history of what you've pointed the agent at, remove this one bind line from the script — login (`.credentials.json`) still persists without it; you'll just re-answer the onboarding and trust prompts each session.

### `--new-session`

```sh
--new-session
```

Detaches the sandboxed process from the controlling terminal's session in a way that blocks `TIOCSTI`-style keystroke injection back into your real terminal — a bubblewrap-recommended flag for any general-purpose sandbox, not specific to Claude Code, but worth having here.

### `DISABLE_AUTOUPDATER`

```sh
--setenv DISABLE_AUTOUPDATER "1"
```

Belt-and-suspenders on top of installing via AUR (which already disables the built-in updater): even if that ever changes upstream, this environment variable stops Claude Code from attempting to replace its own binary. Pacman owns the binary; `paru -Syu` is how it updates (Step 9).

Everything else — `--clearenv`, the five-variable allowlist, `--unshare-all --share-net`, `--die-with-parent`, `--tmpfs "$HOME"` — is identical in purpose to the OpenCode wrapper. See that guide's Step 5/6 for the full flag-by-flag rationale if you want it again.

---

## Step 6 — Enable the built-in sandbox and log in

The `settings.json` from Step 3 already sets `"sandbox": { "enabled": true, ... }`, so the built-in sandbox is active from the first run — no separate `/sandbox` setup step needed the way it would be with a fresh, unconfigured install. You can still run `/sandbox` inside a session to inspect status or check the Dependencies tab (confirms `bubblewrap`, `socat`, and the seccomp filter are all detected).

Log in through the wrapper so the credential lands in the dedicated state directory:

```bash
mkdir -p ~/tmp/cclaude-login
cclaude ~/tmp/cclaude-login
```

Then inside Claude Code:

```text
/login
```

Follow the prompts. Because the sandbox has no display/browser socket exposed (this is a CLI-only setup — no GUI passthrough), the automatic browser launch will not succeed; Claude Code falls back to printing a URL. Copy it, open it in your normal browser, sign in with your Claude Pro/Max/Team/Enterprise account, and paste the confirmation code back into the terminal if prompted. This is the same flow Anthropic documents for headless/remote environments.

On success, the token is written to `.credentials.json` inside `~/.claude/` in the sandbox — on the host, that's:

```bash
ls -la ~/.local/state/claude-code-sandbox/
```

Lock it down:

```bash
chmod 700 ~/.local/state/claude-code-sandbox
chmod 600 ~/.local/state/claude-code-sandbox/.credentials.json 2>/dev/null || true
```

To log out / revoke locally at any time: `rm -rf ~/.local/state/claude-code-sandbox/*` (then restore `settings.json` from Step 3 and `touch` a new `legacy-claude.json`), plus sign the device out from your Anthropic account's security page if you want server-side revocation too.

---

## Step 7 — First run and reviewing changes without git

Same workflow as the OpenCode guide — snapshot, work, diff, decide:

```bash
cd ~/projects/myproject
cp -r ~/projects/myproject ~/projects/myproject.bak
cclaude
```

First launch in a new project directory shows the workspace-trust dialog once — choose "yes, trust this folder." Because the legacy `~/.claude.json` is persisted (Step 5), it won't re-prompt for that same folder on later sessions.

After a session:

```bash
diff -ru ~/projects/myproject.bak ~/projects/myproject
```

Undo everything:

```bash
rm -rf ~/projects/myproject
cp -r ~/projects/myproject.bak ~/projects/myproject
```

Accept everything:

```bash
rm -rf ~/projects/myproject.bak
```

Daily habits:

* With `defaultMode: "default"` and `autoAllowBashIfSandboxed: false`, every tool use and every Bash command prompts. Read them.
* Use Plan Mode (`defaultMode: "plan"` for a session, or the in-session toggle) when you just want analysis with zero file/command changes.
* Snapshot before, `diff -ru` after, every session.

---

## Step 8 — OpenSnitch: network egress control (tested on Artix runit)

This is the same daemon, same runit quirk, same notification-daemon fix as the OpenCode guide — reproduced here so this repo stands alone.

### 8a. Install

```bash
sudo pacman -S --needed opensnitch
```

### 8b. Create the runit service manually

Artix ships an OpenRC unit for `opensnitch`, not a runit one:

```bash
sudo mkdir -p /etc/runit/sv/opensnitchd
sudo mkdir -p /etc/opensnitchd/rules
sudo tee /etc/runit/sv/opensnitchd/run << 'EOF'
#!/bin/sh
exec >>/var/log/opensnitchd.log 2>&1
exec /usr/bin/opensnitchd -rules-path /etc/opensnitchd/rules/
EOF
sudo chmod +x /etc/runit/sv/opensnitchd/run
sudo ln -s /etc/runit/sv/opensnitchd /run/runit/service/
sleep 2
sudo sv status opensnitchd
```

`exec >>/var/log/opensnitchd.log 2>&1` sends the daemon's stdout/stderr to a dedicated log file instead of runit's default log capture — check it with `sudo tail -f /var/log/opensnitchd.log` if something looks wrong. Nothing rotates this file automatically, so make sure your system's log rotation covers `/var/log/*.log`, or it will grow unbounded.

Expect: `run: opensnitchd: (pid NNNN) Ns`.

### 8c. Fix the GUI: install a notification daemon

`opensnitch-ui` crashes on distros with no `org.freedesktop.Notifications` implementation running — looks like a Qt/tray bug, isn't one:

```bash
sudo pacman -S --needed dunst libnotify python-notify2
dunst &
notify-send "OpenSnitch test" "notifications work"
```

Make it permanent via `~/.xprofile` or `~/.xinitrc`:

```bash
if ! pgrep -x dunst >/dev/null; then
    dunst &
fi
```

### 8d. Set default-deny

```bash
opensnitch-ui &
```

Preferences → Nodes → your node → **Default action** for outbound connections → **Deny**. Confirm unmatched connections prompt/deny rather than passing through.

Verify:

```bash
curl https://example.com
```

(as your normal user, outside any sandbox) — should now prompt or fail.

### 8e. Train rules

With `opensnitch-ui` open, approve "always" for your browser and `pacman`/`paru`, scoped to the specific executable. Then run:

```bash
cclaude ~/tmp/cclaude-login
```

and approve, "always," only the domains you recognize — Anthropic's API domains, and whatever the login/OAuth flow needs. Deny the rest.

Verify child-process blocking works even for a sandboxed session:

```text
(inside a cclaude session) run: curl https://example.com
```

Should be denied both by Claude Code's own `sandbox.network.allowedDomains: []` (falls to ask/deny) and, if it somehow got through that, by OpenSnitch underneath.

Close `opensnitch-ui` once rules are stable; `opensnitchd` keeps enforcing in the background.

---

## Step 9 — Safe updating

```bash
paru -Syu
```

Skim the changed PKGBUILD when `claude-code-bin`/`claude-code-stable-bin` updates (source URL, checksums). Never rely on Claude Code's own updater for this install — it's disabled both by the AUR packaging and by `DISABLE_AUTOUPDATER=1` in the wrapper.

Run `claude doctor` occasionally (through the wrapper) to check for configuration or install-method drift:

```bash
cclaude ~/tmp/cclaude-login
```
```text
/doctor
```
(or, if you prefer a raw check outside a session, whatever CLI-level diagnostic flag your version supports)

---

## Step 10 — Test checklist before real projects

```bash
mkdir -p ~/tmp/cclaude-sandbox-test
cd ~/tmp/cclaude-sandbox-test
echo "hello" > README.md
cp -r ~/tmp/cclaude-sandbox-test ~/tmp/cclaude-sandbox-test.bak
cclaude
```

Ask Claude Code to run each of these (approve the Bash prompts to let them execute):

**Test 1 — real home hidden?**
```text
Run: ls -la ~
```
Expected: a nearly empty fake home (`.claude/`, `project/`), nothing personal.

**Test 2 — SSH/AWS/GPG unreachable?**
```text
Run: ls -la ~/.ssh
Run: ls -la ~/.aws
Run: ls -la ~/.gnupg
```
Expected: no such directory, for all three.

**Test 3 — settings.json visible and correct?**
```text
Run: cat ~/.claude/settings.json
```
Expected: your hardened config, sandboxing and deny rules intact.

**Test 4 — settings.json specifically read-only, rest of .claude/ writable?**
```text
Run: echo test >> ~/.claude/settings.json
Run: touch ~/.claude/test-file
```
Expected: first fails (permission denied), second succeeds (Claude Code can still write session/credential state).

**Test 5 — only the project persists?**
```text
Run: touch ./write-test.txt
Run: touch ~/outside-test.txt
```
Expected: both succeed inside the session, but after closing Claude Code, only `write-test.txt` exists on your real disk.

**Test 6 — environment clean?**
```text
Run: env
```
Expected: only HOME, USER, PATH, TERM, LANG, DISABLE_AUTOUPDATER, plus whatever Claude Code itself adds. No leaked shell secrets.

**Test 7 — built-in sandbox actually enforcing?**
```text
Run: touch /tmp/outside-write-attempt
```
(this targets the sandbox's own `/tmp`, not the project — should still succeed since `/tmp` is normally writable; more telling:)
```text
Run: echo test > /etc/test-file
```
Expected: denied — `/etc` (actually the whole non-project filesystem) is not writable even to the Bash tool's own OS-level sandbox.

**Test 8 — random network blocked?**
```text
Run: curl https://example.com
```
Expected: falls through to a permission prompt (due to `allowedDomains: []`) and/or an OpenSnitch prompt. Do not allow it permanently.

**Test 9 — provider works?**
Ask a simple coding question. Expected: responds normally once you've approved Anthropic's API domains.

All nine pass → cleared for real projects.

---

## Step 11 — Daily usage & cheat sheet

| Task | Command |
| --- | --- |
| Session in current project | `cclaude` |
| Session in specific project | `cclaude ~/projects/foo` |
| Log in / re-auth | `cclaude ~/tmp/cclaude-login` then `/login` |
| Snapshot before session | `cp -r ~/projects/foo ~/projects/foo.bak` |
| See agent's changes | `diff -ru ~/projects/foo.bak ~/projects/foo` |
| Undo everything | `rm -rf ~/projects/foo && cp -r ~/projects/foo.bak ~/projects/foo` |
| Log out + wipe all state | `rm -rf ~/.local/state/claude-code-sandbox/*` (then restore settings.json + `touch legacy-claude.json`) |
| Update safely | `paru -Syu` |
| Check sandbox status | inside a session: `/sandbox` |
| Check overall health | inside a session: `/doctor` |

---

## Final security summary

* One writable project directory, enforced by **two independent bubblewrap layers**: the outer wrapper (confines the whole `claude` process) and Claude Code's built-in sandbox (confines the Bash tool specifically, even if the outer wrapper were somehow bypassed)
* Fake home: SSH, GPG, AWS, browser data, and dotfiles all invisible; the legacy trust/onboarding/project-history file is persisted to the state dir by default (removable for zero on-disk project history — see Step 5)
* Zero secrets in the environment (`--clearenv` + five harmless variables + one explicit updater-disable flag)
* Login credential confined to a dedicated, permission-locked, wipeable state directory
* `settings.json` read-only inside the wrapper; `bypassPermissions` disabled at the config level, not just by convention
* `defaultMode: "default"` — every tool use prompts; nothing runs unattended
* Network deny-by-default at three points: sandbox `allowedDomains: []`, the permission system's ask flow, and OpenSnitch underneath both
* Binary owned and verified by pacman/AUR; auto-update disabled two ways
* Every change reviewable and fully undoable via snapshot diff

Remaining risks you accept knowingly: the agent reads everything in the project you mount, and it can read its own login token while running. Keep OpenSnitch on, read permission prompts before approving, and revoke the session from your Anthropic account if anything ever looks wrong.
