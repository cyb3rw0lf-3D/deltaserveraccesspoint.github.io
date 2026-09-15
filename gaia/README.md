# Gaia

Self-hosted AI assistant (Ollama) running on a homelab server, reachable
remotely — along with SSH admin access — through Cloudflare Tunnel +
Cloudflare Access, with tiered privilege on the host (key-only SSH login,
2FA-gated root).

Nothing in this repo runs the AI itself — GitHub Pages is static hosting.
This directory holds the deployment config you pull down and run **on your
own homelab server**. No secrets live here; every credential is created
and stored only on that machine.

See [`DESIGN.md`](DESIGN.md) for the reasoning behind these choices, if
you're picking this up fresh.

## Architecture

```
you --> Cloudflare Access (login) --> Cloudflare Tunnel --> cloudflared (homelab)
                                                               |    |
                                                    Ollama (127.0.0.1) |
                                                              sshd (localhost:22)
```

- **No inbound ports opened on your router/firewall, for either service.**
  `cloudflared` makes an outbound connection to Cloudflare; both Ollama and
  SSH are only reachable through the tunnel, never directly — Ollama is
  bound to `127.0.0.1`, and SSH is never port-forwarded on the router.
- **Cloudflare Access** sits in front of *each* tunnel hostname separately
  and requires you to authenticate (email OTP, SSO, whatever you
  configure) before any request reaches the homelab box at all — this
  applies independently to the AI hostname and the SSH hostname, so you
  can set different policies for each if you want.
- **Host-level hardening** (key-only login, non-root, 2FA-gated root) is
  independent of Cloudflare and stays in place underneath it — Access is
  an *additional* login layer in front of SSH, not a replacement for it.
  See steps 6–7 below.
- Any device, anywhere, can reach either service once this is set up —
  it just needs `cloudflared` (for SSH) or a browser (for the AI) and to
  pass the Access login. No per-device "hookup" beyond that.

## 1. Install prerequisites (on the homelab server)

```bash
# Docker + Compose plugin (Debian/Ubuntu example)
curl -fsSL https://get.docker.com | sh

# cloudflared
curl -fsSL https://pkg.cloudflare.com/cloudflared-stable-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
```

## 2. Run Ollama

```bash
cd gaia
docker compose up -d
# GPU host:
# docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

Pull a model once it's up:

```bash
docker exec -it gaia-ollama ollama pull llama3.1
```

Ollama is bound to `127.0.0.1:11434` — it is not reachable from your LAN
or the internet directly. It's only reachable through the tunnel below.

## 3. Create the Cloudflare Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create gaia
cloudflared tunnel route dns gaia gaia.android21engine.org
cloudflared tunnel route dns gaia ssh.android21engine.org
```

Copy `cloudflared/config.yml.example` to `cloudflared/config.yml`, fill in
the tunnel ID and hostname it printed, then run it:

```bash
cp cloudflared/config.yml.example cloudflared/config.yml
# edit cloudflared/config.yml
cloudflared tunnel run gaia
```

(Set it up as a systemd service — `cloudflared service install` — so it
survives reboots.)

`cloudflared/config.yml` and the tunnel credentials JSON are gitignored.
Never commit them.

## 4. Require login before traffic reaches the tunnel: Cloudflare Access

In the Cloudflare Zero Trust dashboard, add **two Access applications**,
one per hostname:

- `gaia.android21engine.org` (type: Self-hosted) — gates the AI.
- `ssh.android21engine.org` (type: Self-hosted, or "SSH" if offered) —
  gates administrative access to the box.

Set a policy on each for who may authenticate (your email, a group, SSO
provider, etc.) — they can be the same policy or different ones. Once
these are in place, nobody reaches Ollama or SSH without passing that
login first — this is the "login" layer; it lives in Cloudflare because
GitHub Pages has no backend to run it.

## 5. Connect to SSH through the tunnel, from any device

On whatever device you're connecting *from* (not the homelab box), install
`cloudflared` and add this to `~/.ssh/config`:

```
Host gaia-ssh
  HostName ssh.android21engine.org
  ProxyCommand cloudflared access ssh --hostname %h
  User <your-non-root-username>
```

Then `ssh gaia-ssh` — it opens a browser for the Access login, then drops
you into a normal SSH session. No port was ever opened on the router; the
connection goes out through the same tunnel as the AI traffic.

## 6. Harden host (SSH) access, independently of Cloudflare

Cloudflare Access is a login gate *in front of* SSH, not a substitute for
securing SSH itself. This governs administrative access to the box itself,
not the AI service, and applies whether you connect via the tunnel above
or directly on the LAN.

```bash
# Disable password auth, key-only login (edit /etc/ssh/sshd_config):
#   PasswordAuthentication no
#   PermitRootLogin no
sudo systemctl restart sshd
```

Log in as a normal, non-root user with an SSH key. Root is never reached
directly — only via `sudo` from that account.

## 7. Gate root (`sudo`) behind a second factor

```bash
sudo apt install libpam-google-authenticator
google-authenticator   # run as the non-root user, scan the QR code
```

Add to `/etc/pam.d/sudo`:

```
auth required pam_google_authenticator.so
```

Now `sudo` prompts for a TOTP code in addition to the user's normal auth —
root is never reachable by a password (or key) alone.

## Secrets checklist

Never commit any of the following — all are already covered by
`gaia/.gitignore`:

- `cloudflared/config.yml` (the real one — only `.example` is tracked)
- Tunnel credentials JSON (`cloudflared/<tunnel-id>.json`)
- Any `.env` file
- TOTP secrets / QR codes
