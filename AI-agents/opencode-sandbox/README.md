# opencode-artix-sandbox

A hardened, sandboxed setup for running [OpenCode](https://opencode.ai) (an
AI coding agent) on Artix Linux — TUI, web GUI, no separate API billing
(uses OpenCode's own provider login instead of API keys), and no systemd
dependency anywhere in the stack.

## What this is

OpenCode is a capable but unrestricted agent: by default it can read/write
any file it has permission to touch and execute arbitrary shell commands.
This repo sandboxes it with [bubblewrap](https://github.com/containers/bubblewrap)
so that:

- Only **one project directory** is writable at a time
- Your real home, SSH keys, GPG keys, browser data, and shell history are
  **invisible** inside the sandbox (fake `tmpfs` home)
- The shell environment is **cleared and rebuilt** from an explicit
  allowlist — no leaked tokens or credentials
- OpenCode's own config (`opencode.json`) is bind-mounted so its policy
  (sharing off, autoupdate off, permission prompts on) can't be
  self-modified by the agent
- [OpenSnitch](https://github.com/evilsocket/opensnitch) enforces
  deny-by-default network egress, including for child processes the agent
  spawns (`curl`, `node`, `python`, etc.)

Written and tested on **Artix Linux with runit**, but the bubblewrap parts
apply to any Linux distro; only Step 9 (service supervision) is
runit/Artix-specific.

## Repo layout

```
.
├── README.md                  — you are here
├── docs/
│   └── hardened-setup.md      — the full step-by-step guide (start here)
├── scripts/
│   ├── ocode                  — the sandbox launcher script
│   └── opensnitchd-run        — runit run-script for opensnitchd
└── config/
    └── opencode.json          — hardened OpenCode config (sharing/autoupdate off, ask-permissions)
```

## Quick start

Read [`docs/hardened-setup.md`](docs/hardened-setup.md) in full before
running anything — it explains every command and every sandbox flag. The
short version:

```bash
# 1. Install
sudo pacman -S --needed bubblewrap opensnitch git
# (see docs for AUR/paru install of opencode-bin)

# 2. Config
mkdir -p ~/.config/opencode
cp config/opencode.json ~/.config/opencode/opencode.json

# 3. Launcher
mkdir -p ~/.local/bin ~/.local/state/opencode-sandbox
cp scripts/ocode ~/.local/bin/ocode
chmod +x ~/.local/bin/ocode
export PATH="$HOME/.local/bin:$PATH"   # add to ~/.bashrc to persist

# 4. OpenSnitch (Artix/runit) — see docs Step 9 for the notification-daemon fix too
sudo mkdir -p /etc/runit/sv/opensnitchd /etc/opensnitchd/rules
sudo cp scripts/opensnitchd-run /etc/runit/sv/opensnitchd/run
sudo chmod +x /etc/runit/sv/opensnitchd/run
sudo ln -s /etc/runit/sv/opensnitchd /run/runit/service/

# 5. Log in (no API key needed) and go
ocode ~/tmp/ocode-login auth login
ocode ~/projects/myproject
```

## Security model / honest limits

This sandbox reduces blast radius; it does not make the agent harmless.
Inside the sandbox, the agent can still:

- read/write everything in the one mounted project directory
- read its own provider login token while running

Mitigations for those: don't mix secrets into project directories, review
every `bash`/`webfetch` permission prompt, keep OpenSnitch's default-deny
active, and revoke the provider session if anything looks wrong. Full
details, a test checklist, and daily-use commands are in
[`docs/hardened-setup.md`](docs/hardened-setup.md).

## License

MIT — see [`LICENSE`](LICENSE). Use at your own risk; review every script
before running it, especially anything invoked with `sudo`.
