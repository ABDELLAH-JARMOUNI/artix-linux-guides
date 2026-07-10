# claude-code-sandbox

A hardened, sandboxed setup for running [Claude Code](https://code.claude.com)
on Artix Linux — CLI only, no separate API billing (uses your Claude
Pro/Max subscription login instead of an API key), no systemd dependency
anywhere in the stack.

## What makes this different from a typical agent sandbox

Claude Code ships its **own OS-level sandbox** (bubblewrap on Linux, the
same primitive this repo uses for the outer wrapper). Anthropic's docs
describe permissions and sandboxing as complementary layers meant to be
combined, not alternatives — so this setup runs three layers instead of
one:

1. **Claude Code's built-in sandbox** — OS-level filesystem/network isolation for the Bash tool and everything it spawns (`sandbox` block in `settings.json`)
2. **An outer bubblewrap wrapper** (`cclaude`) — confines the whole `claude` process from launch: fake home, cleared environment, one writable project directory
3. **OpenSnitch** — deny-by-default network egress underneath both

Written and tested on **Artix Linux with runit**; the bubblewrap parts
apply to any Linux distro, and only the OpenSnitch service-file step is
runit/Artix-specific.

## Repo layout

```
.
├── README.md                  — you are here
├── docs/
│   └── hardened-setup.md      — the full step-by-step guide (start here)
├── scripts/
│   ├── cclaude                — the outer sandbox launcher script
│   └── opensnitchd-run        — runit run-script for opensnitchd
└── config/
    └── settings.json          — hardened Claude Code config (sandbox on, ask-permissions, bypass disabled)
```

## Quick start

Read [`docs/hardened-setup.md`](docs/hardened-setup.md) in full first — it
explains every command and every flag, including how the two sandbox
layers interact. Short version:

```bash
# 1. Install
sudo pacman -S --needed bubblewrap socat opensnitch git
# claude-code itself: via AUR (see docs for PKGBUILD review steps)
paru -S claude-code-bin

# 2. Config — note this goes in the *state dir*, not ~/.claude directly,
#    because the wrapper replaces $HOME entirely
mkdir -p ~/.local/state/claude-code-sandbox
cp config/settings.json ~/.local/state/claude-code-sandbox/settings.json
touch ~/.local/state/claude-code-sandbox/legacy-claude.json  # persists onboarding/trust/history

# 3. Launcher
mkdir -p ~/.local/bin
cp scripts/cclaude ~/.local/bin/cclaude
chmod +x ~/.local/bin/cclaude
export PATH="$HOME/.local/bin:$PATH"   # add to ~/.bashrc to persist

# 4. OpenSnitch (Artix/runit) — see docs Step 8 for the notification-daemon fix too
sudo mkdir -p /etc/runit/sv/opensnitchd /etc/opensnitchd/rules
sudo cp scripts/opensnitchd-run /etc/runit/sv/opensnitchd/run
sudo chmod +x /etc/runit/sv/opensnitchd/run
sudo ln -s /etc/runit/sv/opensnitchd /run/runit/service/

# 5. Log in (no API key needed) and go
mkdir -p ~/tmp/cclaude-login
cclaude ~/tmp/cclaude-login   # then run /login inside
cclaude ~/projects/myproject
```

## Security model / honest limits

This sandbox reduces blast radius; it does not make the agent harmless.
Inside it, the agent can still:

- read/write everything in the one mounted project directory
- read its own login credential while running

Mitigations: don't mix secrets into project directories, read every
permission prompt before approving (the config ships with
`defaultMode: "default"` and `autoAllowBashIfSandboxed: false` — nothing
runs unattended), keep OpenSnitch's default-deny active, and revoke the
session from your Anthropic account if anything looks wrong. Full
details and a 9-point test checklist are in
[`docs/hardened-setup.md`](docs/hardened-setup.md).

## License

MIT — see [`LICENSE`](LICENSE). Use at your own risk; review every script
before running it, especially anything invoked with `sudo`.
