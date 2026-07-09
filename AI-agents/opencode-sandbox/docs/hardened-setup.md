# OpenCode on Artix — Hardened Sandbox Setup (No API Keys Edition)

Tailored to your case: **your own local projects**, TUI as the daily driver, web GUI when needed, **no separate API billing** — you log in to your provider account through OpenCode's own auth instead of pasting API keys. Opsec-first.

The hardened setup is:

* OpenCode installed from Arch/AUR package manager, not from a curl installer
* No API keys anywhere — provider login handled by `opencode auth login` inside the sandbox
* Login credentials stored **only** in a dedicated sandboxed state directory, never in your real home
* A launcher script `ocode` that runs OpenCode inside a bubblewrap sandbox
* Only the current project directory is writable inside the sandbox
* Your real home directory, SSH keys, GPG keys, browser profiles, and shell history are invisible inside the sandbox
* OpenCode config bind-mounted read-only so sharing, autoupdate, and permissions are enforced
* Inherited shell environment cleared — nothing leaks in
* OpenSnitch controls outbound network access

The desktop app is out of scope for this setup — TUI and web GUI cover everything with a much cleaner security model.

---

## Step 0 — The security model and its limits

What this gives you:

* Filesystem blast radius: one project directory
* Environment blast radius: five harmless variables (HOME, USER, PATH, TERM, LANG) — zero secrets injected
* Credential blast radius: OpenCode's login tokens exist only inside the dedicated sandbox state dir
* Network blast radius: only explicitly approved provider endpoints
* Agent changes: reviewable with a backup-copy diff (or git later)
* Permissions: ask before edits, shell commands, web fetches
* Sharing: disabled. Self-update: disabled.

Honest limits — read this part:

* The agent **can** read everything in the mounted project directory. Don't point it at folders containing secrets, credentials, or private documents mixed in with code.
* The agent **can** read its own login token in the sandbox state dir (`auth.json` is plaintext on disk). If that token leaks, someone can use your provider account — so OpenSnitch egress control and reviewing permission prompts still matter.
* The sandbox stops file theft and file damage outside the project. It does not make the agent smart or honest. Review what it does.

---

## Step 1 — Install base packages

```bash
sudo pacman -S --needed bubblewrap git
```

Explained:

* `sudo pacman -S` installs packages; `--needed` skips what's already present.
* `bubblewrap` provides `bwrap`, the sandbox for the TUI and web GUI.
* `git` is needed to build AUR packages (paru). You are not required to use it for your projects.

If you do not have an AUR helper yet, install `paru`:

```bash
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin
less PKGBUILD
makepkg -si
cd .. && rm -rf paru-bin
```

Explained:

* `git clone` downloads the AUR build recipe (not paru itself — the instructions to build it).
* `less PKGBUILD` — read it before building. Check `source=` points to the official paru GitHub, checksums are not `SKIP`. Press `q` to exit.
* `makepkg -si` builds as your normal user (`-s` pulls build deps, `-i` installs the result).
* The last line removes the build directory.

---

## Step 2 — Install OpenCode

```bash
paru -S opencode-bin
```

When paru shows the PKGBUILD, verify:

1. `source=` URLs point to the official OpenCode GitHub releases
2. `sha256sums=` contains real checksums, not `SKIP`
3. no odd extra commands in `prepare()` / `package()`

Confirm it runs:

```bash
opencode --version
```

Do **not** use the curl installer (`curl ... | bash`) — piping remote scripts into your shell is the exact pattern this guide exists to avoid, and pacman should own the binary so updates stay verified.

---

## Step 3 — Hardened OpenCode config

```bash
mkdir -p ~/.config/opencode
cat > ~/.config/opencode/opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "share": "disabled",
  "autoupdate": false,
  "permission": {
    "edit": "ask",
    "bash": "ask",
    "webfetch": "ask"
  },
  "server": {
    "hostname": "127.0.0.1",
    "mdns": false
  }
}
EOF
```

Explained:

* `mkdir -p` — create the config directory (no error if it exists).
* `cat > file << 'EOF' ... EOF` — a heredoc: writes everything between the markers into the file; the quoted `'EOF'` prevents the shell from expanding anything inside.
* `"$schema"` — enables editor validation; no runtime effect.
* `"share": "disabled"` — kills the `/share` feature, which would upload session transcripts (your code + conversation) to OpenCode's servers. Must be off for "code stays local except the model API."
* `"autoupdate": false` — OpenCode must not replace its own binary; pacman owns it.
* `"permission"` — ask before file edits, shell commands, and web fetches. This is your review checkpoint on everything the agent does.
* `"server"` — web GUI binds to localhost only, no LAN advertising via mDNS.

Version note: config keys change between OpenCode releases. On first run, watch for "unknown key" warnings; if a key is rejected (the `server` block is the most likely candidate), remove it — the sandbox does not depend on any of these lines, they are defense-in-depth on top of it.

Critical detail: this config only takes effect because the launcher below bind-mounts `~/.config/opencode` into the sandbox's fake home. Never remove that bind.

---

## Step 4 — The sandbox launcher

Create the directories:

```bash
mkdir -p ~/.local/bin ~/.local/state/opencode-sandbox
```

Explained:

* `~/.local/bin` — where the launcher lives (XDG-standard; usually already in PATH).
* `~/.local/state/opencode-sandbox` — the **dedicated state directory**. OpenCode's login credentials (`auth.json`), sessions, and history live here and only here — never in your real config, wipeable at any time.

Create the launcher:

```bash
cat > ~/.local/bin/ocode << 'EOF'
#!/bin/sh
# ocode — run OpenCode sandboxed in the current directory or a given project directory
# usage:
#   ocode
#   ocode /path/to/project
#   ocode /path/to/project web
#   ocode . run "explain the build system"
#   ocode . auth login

set -eu

if [ -d "${1:-}" ]; then
  PROJECT="$(realpath "$1")"
  shift
else
  PROJECT="$(realpath .)"
fi

STATE="$HOME/.local/state/opencode-sandbox"

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
  --dir "$HOME/.config" \
  --bind "$HOME/.config/opencode" "$HOME/.config/opencode" \
  --ro-bind "$HOME/.config/opencode/opencode.json" "$HOME/.config/opencode/opencode.json" \
  --dir "$HOME/.local" \
  --dir "$HOME/.local/share" \
  --dir "$HOME/.cache" \
  --bind "$PROJECT" "$HOME/project" \
  --bind "$STATE" "$HOME/.local/share/opencode" \
  --unshare-all \
  --share-net \
  --die-with-parent \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv USER "${USER:-user}" \
  --setenv PATH "/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  --chdir "$HOME/project" \
  opencode "$@"
EOF
chmod +x ~/.local/bin/ocode
```

Check `~/.local/bin` is in your PATH:

```bash
echo "$PATH"
```

If missing, add it to your shell's startup file and reload the shell. For bash, that's `~/.bashrc`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

(If you use a different shell, use its equivalent file — `~/.zshrc` for zsh, `~/.profile` for a login-shell-wide setting.)

---

## Step 5 — Launcher explained, line by line

### Project selection

```sh
if [ -d "${1:-}" ]; then
  PROJECT="$(realpath "$1")"
  shift
else
  PROJECT="$(realpath .)"
fi
```

* No argument → current directory is the project.
* First argument is a directory → that's the project; remaining args pass through to OpenCode.
* `set -eu` at the top makes the script abort loudly on any error or unset variable.
* `realpath` resolves the absolute path so the bind mount is unambiguous.

### Read-only system

```sh
--ro-bind /usr /usr
```

The sandbox uses your system's binaries and libraries — your real compilers, node, python — but cannot modify them. The `--symlink` lines recreate Arch's `/lib → usr/lib` layout so paths resolve.

### Minimal /etc

```sh
--ro-bind /etc/resolv.conf ...
--ro-bind /etc/ssl ...
--ro-bind /etc/ca-certificates ...
```

DNS and TLS certificates only — the rest of `/etc` (which can contain things the agent has no business reading) does not exist inside. If DNS ever misbehaves in the sandbox, add `--ro-bind /etc/hosts /etc/hosts` next to these.

### Private /proc, /dev, /tmp

```sh
--proc /proc
--dev /dev
--tmpfs /tmp
```

Fresh process view, minimal devices, and a private in-RAM `/tmp` that vanishes on exit.

### Fake home — the core of the sandbox

```sh
--dir /home
--tmpfs "$HOME"
```

Your home directory is replaced by an empty in-RAM one. Invisible inside the sandbox:

* `~/.ssh`
* `~/.gnupg`
* browser profiles
* shell history
* all your dotfiles and documents

### Config: writable directory, locked policy file

```sh
--bind "$HOME/.config/opencode" "$HOME/.config/opencode"
--ro-bind "$HOME/.config/opencode/opencode.json" "$HOME/.config/opencode/opencode.json"
```

OpenCode needs to write small housekeeping files into its own config folder (a `.gitignore`, cache markers, etc.) — a fully read-only directory makes it error out (`FileSystem.writeFile`). So the directory itself is mounted read-write, but `opencode.json` is then re-mounted read-only **on top of it** — bwrap applies binds in order, so the later, more specific bind wins for that one file. Net effect: OpenCode can create whatever files it needs alongside the config, but it cannot modify the policy file itself — sharing, autoupdate, and permission settings stay locked regardless of what the agent tries. Settings changed in the TUI still won't persist to `opencode.json`; edit it on the host if you want to change policy.

### Dedicated state — where your login lives

```sh
--bind "$STATE" "$HOME/.local/share/opencode"
```

This is the **only** writable location besides the project. When you log in (Step 6), OpenCode writes its credentials to `auth.json` here — meaning on the host, your provider login exists only at:

```text
~/.local/state/opencode-sandbox/
```

Never in your real `~/.local/share`. Wipe it anytime to log out everywhere and destroy all sessions:

```bash
rm -rf ~/.local/state/opencode-sandbox/*
```

### Writable project

```sh
--bind "$PROJECT" "$HOME/project"
```

The main security boundary: the one read-write mount. The full blast radius of anything the agent does to disk.

### Namespaces and network

```sh
--unshare-all
--share-net
```

New PID/IPC/user/hostname namespaces — the agent can't see or signal your other processes. Networking is deliberately shared back because the model API needs it; **filtering** the network is OpenSnitch's job (Step 8).

### Cleared environment

```sh
--clearenv
--setenv HOME "$HOME"
--setenv USER "${USER:-user}"
--setenv PATH "/usr/local/bin:/usr/bin:/bin"
--setenv TERM "${TERM:-xterm-256color}"
--setenv LANG "${LANG:-C.UTF-8}"
```

Do not skip `--clearenv`. Without it, bwrap passes your **entire** shell environment into the sandbox — any `GITHUB_TOKEN`, cloud credentials, proxy passwords, or other secrets you've ever exported. With it, the environment is rebuilt from this five-variable allowlist and carries **zero secrets** — this edition of the setup injects no API keys at all.

### Safety net

```sh
--die-with-parent
```

If your terminal dies, the sandboxed agent dies with it. No orphaned processes.

---

## Step 6 — Log in to your provider (inside the sandbox)

No API keys, no billing setup — you authenticate with your existing provider account (e.g. your ChatGPT/Claude subscription login) through OpenCode's auth flow. Crucially, do this **through the sandbox** so the credentials land in the dedicated state dir:

```bash
mkdir -p ~/tmp/ocode-login
ocode ~/tmp/ocode-login auth login
```

Explained:

* We use a throwaway directory because `ocode` needs some project to mount; the login itself is project-independent.
* `auth login` is forwarded to OpenCode inside the sandbox. Pick your provider from the list and follow the flow — typically it prints a URL and a code; open the URL in your normal browser, log in to your provider account, and enter/confirm the code.
* When it completes, the token is written to `auth.json` inside the sandbox state dir — on the host that's `~/.local/state/opencode-sandbox/`, **not** your real home.

Verify where the credential landed:

```bash
ls -la ~/.local/state/opencode-sandbox/
```

You should see `auth.json` (and possibly other state). Lock its permissions down:

```bash
chmod 700 ~/.local/state/opencode-sandbox
chmod 600 ~/.local/state/opencode-sandbox/auth.json 2>/dev/null || true
```

Explained:

* `chmod 700` on the directory — only your user can enter it.
* `chmod 600` on the file — only your user can read it. The `|| true` keeps the command from failing if the filename differs on your version.

Opsec notes on this login:

* The token in `auth.json` is plaintext on disk. It is protected by file permissions and by living outside the sandbox's reach only when the sandbox isn't running — the agent itself **can** read its own token while running. That's unavoidable (it needs the token to talk to the API); the mitigations are OpenSnitch (the token can't be *sent* anywhere except approved endpoints) and easy revocation.
* To log out / revoke locally: `rm -rf ~/.local/state/opencode-sandbox/*`, and also sign out the device/session from your provider's account security page.
* During login, OpenSnitch (once installed) will prompt for the provider's **auth** domains, not just the API domain — approve them for the login, and you can set those rules to expire rather than "always."

---

## Step 7 — First run and reviewing changes without git

Go to a project and snapshot it first:

```bash
cd ~/projects/myproject
cp -r ~/projects/myproject ~/projects/myproject.bak
```

Explained:

* `cp -r` recursively copies the whole project. `myproject.bak` is your untouched pre-agent snapshot.

Run OpenCode:

```bash
ocode
```

Inside the TUI, initialize the project once:

```text
/init
```

This creates `AGENTS.md` — the agent's notes about your project structure. Open it in any editor and review it.

After any session, see exactly what the agent changed:

```bash
diff -ru ~/projects/myproject.bak ~/projects/myproject
```

* `diff -ru` recursively compares the two trees line by line. Everything it prints is an agent change.

Undo everything:

```bash
rm -rf ~/projects/myproject
cp -r ~/projects/myproject.bak ~/projects/myproject
```

Accept everything (delete the snapshot):

```bash
rm -rf ~/projects/myproject.bak
```

Take a fresh snapshot before each session for per-session undo. This is weaker than git (no history, no partial review) but gives you real "what changed" and "undo all" without learning git. If you adopt git later, `git diff` / `git restore` replace this workflow.

Daily habits:

* **Tab** in the TUI switches `build` (read/write/execute) ↔ `plan` (read-only). Explore in `plan`; switch to `build` only to make changes.
* Keep permission prompts on. Never enable auto-approve.
* Snapshot before, `diff -ru` after. Every session.

---

## Step 8 — Web GUI through the same sandbox

```bash
ocode ~/projects/myproject web
```

Explained: the launcher forwards `web` to OpenCode, which starts the local server + web UI inside the same sandbox and prints a `http://127.0.0.1:PORT` URL — open it in your browser. Because the sandbox shares your network namespace, localhost inside is localhost outside: the browser connects, the agent stays confined to the project directory.

Rules:

* Never bind to `0.0.0.0`, never enable mDNS, never port-forward it. This server reads/writes files and runs shell commands — an exposed port is remote code execution on your machine.
* Need it from another device? SSH tunnel only:

```bash
ssh -L 4096:localhost:4096 you@yourbox
```

(Adjust the port to what OpenCode printed.)

* `Ctrl-C` in the launching terminal stops everything; `--die-with-parent` guarantees no leftovers.

---

## Step 9 — OpenSnitch: network egress control (tested on Artix runit)

Bubblewrap controls files. OpenSnitch controls what can leave your machine — without it, a prompt-injected agent could send data anywhere it wants over the network the sandbox still shares.

### 9a. Install

```bash
sudo pacman -S --needed opensnitch
```

### 9b. Create the runit service manually

Artix's `opensnitch` package ships an OpenRC service, not a runit one — there is no `opensnitchd-runit` package, so `/etc/runit/sv/opensnitchd` does not exist until you create it:

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

Explained:

* `run` is the script runit supervises directly. `exec 2>&1` merges stderr into stdout so daemon errors show up in runit's log rather than vanishing. The final `exec` replaces the shell with the daemon itself so runit tracks the right PID.
* `-rules-path /etc/opensnitchd/rules/` is where allow/deny rules get written; `mkdir -p` ensures it exists before the daemon looks for it.
* The symlink into `/run/runit/service/` activates the service under the current runlevel — same pattern as any other Artix runit service.
* `sleep 2` gives runit's scanner (`runsvdir`) a moment to notice the new service before you check its status; checking immediately can show a false "supervise/ok: file does not exist" error.

Expected result:

```text
run: opensnitchd: (pid NNNN) Ns
```

This is now a real supervised service — it starts automatically on every boot from here on.

### 9c. Fix the GUI: it needs a notification daemon, not just a display

`opensnitch-ui` is a Qt app, but it also tries to send desktop notifications through the freedesktop notification spec. If nothing on your system implements that spec, the GUI crashes with errors like `notify2.UninittedError` or `WARNING: system tray not available` — this looks like a tray/Qt problem but is actually a missing notification service. Fix:

```bash
sudo pacman -S --needed dunst libnotify python-notify2
dunst &
notify-send "OpenSnitch test" "notifications work"
```

Explained:

* `libnotify` provides `notify-send` and the client library `opensnitch-ui` links against.
* `dunst` is the lightweight notification daemon that actually implements `org.freedesktop.Notifications` — without some daemon like this running, any app that tries to notify you will error out.
* `python-notify2` is the Python binding OpenSnitch's GUI uses to talk to that notification service.
* The `notify-send` test line should pop up a small notification bubble — if it does, the notification stack is fixed before you even touch OpenSnitch again.

Make it permanent so you don't have to remember `dunst &` every login. Add to `~/.xprofile` or `~/.xinitrc` (whichever your session actually sources):

```bash
if ! pgrep -x dunst >/dev/null; then
    dunst &
fi
```

### 9d. Understand the three moving parts

This setup has three independent pieces, and it's easy to assume one running means they all are:

| Component | Role | If it's not running |
| --- | --- | --- |
| `opensnitchd` | The actual firewall/monitor/enforcer (runs as root, supervised by runit) | No protection at all, rules or not |
| `opensnitch-ui` | The popup window + rule editor | Daemon still enforces existing saved rules, but you get no interactive prompts for *new*, unmatched traffic |
| `dunst` | Desktop notification service | GUI popups/notifications may not display or the GUI may crash on startup |

Practical consequence: `opensnitch-ui` only needs to be **open** while you're training rules on new traffic (like a fresh OpenCode session hitting a provider domain for the first time). Once your rules are saved, you can close the GUI — `opensnitchd` keeps enforcing them in the background regardless.

### 9e. Set default-deny — the setting that actually matters

Open the GUI:

```bash
opensnitch-ui &
```

In **Preferences** → **Nodes** → your node (usually `localhost`) → find **Default action** for outbound connections and set it to **Deny**. Also check for an "intercept/ask on unknown connections" toggle and make sure it's on.

This is the single most important setting in the whole guide. Fresh OpenSnitch installs commonly default to allowing everything and only logging it — if you skip this step, every rule you write afterward is decorative, since unmatched traffic sails through silently instead of prompting or being denied.

### 9f. Verify default-deny is actually active

Outside any sandbox, as your normal user:

```bash
curl https://example.com
```

Expected: a prompt, or an outright failure. If it succeeds silently, the default-action setting didn't take — go back to 9e.

### 9g. Train rules on everyday traffic

With `opensnitch-ui` open, use your system normally for a bit and approve, "always," scoped to the specific executable:

* your browser(s)
* `pacman` / `paru`
* anything else routine you recognize

Deny anything unfamiliar, or allow it "once" only until you understand what it is.

### 9h. Daily workflow for OpenCode specifically

Before an `ocode` session where you expect new domains (first login, a new provider, a project that fetches new URLs), open the notification stack and the GUI:

```bash
dunst &            # if not already autostarted
opensnitch-ui &
```

Then run your session as normal:

```bash
ocode .
```

When OpenCode (or a child process it spawns) tries to reach a new destination, you'll get a popup — choose **allow only this destination/host**, scoped to the specific executable, rather than a broad allow. Deny unknown domains.

Once your rules are trained and stable, you can close `opensnitch-ui` for day-to-day use — the daemon keeps enforcing everything already approved. Reopen it only when training something new.

Check the daemon is alive anytime with:

```bash
sudo sv status opensnitchd
```

Good output: `run: opensnitchd: ...`

---

## Step 10 — OpenSnitch rule strategy

With default-deny already set (Step 9e) and `opensnitch-ui` open (Step 9h), train the rules in a throwaway project:

```bash
mkdir -p ~/tmp/ocode-test
cd ~/tmp/ocode-test
ocode
```

Approve consciously, per-domain, "always" only for things you recognize:

* your provider's API endpoint (e.g. `api.openai.com` or `api.anthropic.com`, depending on what you logged into)
* `models.dev` — OpenCode fetches provider/model metadata from it
* the provider's auth domains during login (Step 6) — these can be time-limited rules

Deny everything you don't recognize.

Critical: rules scoped to the `opencode` executable are **not enough**. The agent spawns child processes — `curl`, `wget`, `node`, `python`, `git`, package managers, your test scripts — and each is a different executable. Deny-by-default is what actually covers them.

Verify child-process blocking:

```bash
ocode . run "curl https://example.com"
```

This **must** fail or trigger a deny prompt. If it succeeds silently, your policy is too permissive — fix it and retest. Repeat this test after any rule change.

---

## Step 11 — Safe updating

```bash
paru -Syu
```

When OpenCode updates, skim the changed PKGBUILD (source URL, checksums). Never run `opencode upgrade` — pacman owns the binary, and your config already sets `"autoupdate": false`.

---

## Step 12 — Test checklist before real projects

Create a throwaway project:

```bash
mkdir -p ~/tmp/ocode-sandbox-test
cd ~/tmp/ocode-sandbox-test
echo "hello" > README.md
cp -r ~/tmp/ocode-sandbox-test ~/tmp/ocode-sandbox-test.bak
ocode
```

Then ask OpenCode to run each command, and check the result:

**Test 1 — real home hidden?**

```text
Run: ls -la ~
```

Expected: a nearly empty fake home. No `.ssh`, no `.gnupg`, no browser profiles, no personal files.

**Test 2 — SSH keys unreachable?**

```text
Run: ls -la ~/.ssh
```

Expected: no such directory.

**Test 3 — config visible and correct?**

```text
Run: cat ~/.config/opencode/opencode.json
```

Expected: your hardened config, with sharing and autoupdate off.

**Test 4 — policy file specifically read-only, rest of the config dir writable?**

```text
Run: echo test >> ~/.config/opencode/opencode.json
Run: touch ~/.config/opencode/some-other-file
```

Expected: the first command fails (permission denied) — `opencode.json` cannot be modified. The second succeeds — OpenCode can still write housekeeping files into the directory (this is what fixes the `FileSystem.writeFile` error seen during login/init).

**Test 5 — project writable, and only the project persistent?**

```text
Run: touch ./write-test.txt
Run: touch ~/outside-test.txt
```

Expected: both succeed (the second lands in the throwaway in-RAM home), but after closing OpenCode, only `write-test.txt` exists on your real disk — check that `~/outside-test.txt` does **not** exist in your real home.

**Test 6 — environment clean?**

```text
Run: env
```

Expected: only HOME, USER, PATH, TERM, LANG and OpenCode's own additions. No tokens, no secrets, nothing from your shell.

**Test 7 — random network blocked?**

```text
Run: curl https://example.com
```

Expected: blocked or a deny prompt from OpenSnitch. Do not allow it permanently.

**Test 8 — provider works?**

Ask a simple coding question. Expected: the model responds after you've approved the provider endpoints.

All eight pass → cleared for real projects.

---

## Step 13 — Daily usage & cheat sheet

| Task | Command |
| --- | --- |
| TUI in current project | `ocode` |
| TUI in specific project | `ocode ~/projects/foo` |
| One-shot prompt | `ocode . run "explain the build system"` |
| Web GUI | `ocode ~/projects/foo web` |
| Log in / re-auth | `ocode ~/tmp/ocode-login auth login` |
| Snapshot before session | `cp -r ~/projects/foo ~/projects/foo.bak` |
| See agent's changes | `diff -ru ~/projects/foo.bak ~/projects/foo` |
| Undo everything | `rm -rf ~/projects/foo && cp -r ~/projects/foo.bak ~/projects/foo` |
| Undo last change (in TUI) | `/undo` |
| Log out + wipe all agent state | `rm -rf ~/.local/state/opencode-sandbox/*` |
| Update safely | `paru -Syu` |

---

## Final security summary

* One writable project directory — the entire filesystem blast radius
* Fake home: SSH, GPG, browser data, dotfiles, history invisible
* Zero secrets in the environment (`--clearenv` + five harmless variables)
* Provider login token confined to a dedicated, permission-locked, wipeable state dir
* Config read-only inside the sandbox: sharing off, autoupdate off, permission prompts enforced
* Web GUI localhost-only, same sandbox as the TUI
* Network deny-by-default via OpenSnitch, verified against child processes
* Every change reviewable and fully undoable via snapshot diff
* Binary owned and verified by pacman/AUR

Remaining risks you accept knowingly: the agent reads everything in the project you mount, and it can read its own login token while running. Keep OpenSnitch on, review permission prompts, and revoke the provider session if anything ever looks wrong.
