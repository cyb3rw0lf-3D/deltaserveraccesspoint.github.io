# Gaia

Self-hosted AI assistant (Ollama) running on a homelab server, reachable
remotely through Cloudflare Tunnel + Cloudflare Access, with tiered
privilege on the host (key-only SSH login, 2FA-gated root).

Nothing in this repo runs the AI itself — GitHub Pages is static hosting.
This directory holds the deployment config you pull down and run **on your
own homelab server**. No secrets live here; every credential is created
and stored only on that machine.

## Architecture

```
you --> Cloudflare Access (login) --> Cloudflare Tunnel --> cloudflared (homelab)
                                                               |
                                                          Ollama (127.0.0.1 only)
```

- **No inbound ports opened on your router/firewall.** `cloudflared` makes
  an outbound connection to Cloudflare; Ollama itself is bound to
  `127.0.0.1` and is never reachable directly, not even on your LAN.
- **Cloudflare Access** sits in front of the tunnel hostname and requires
  you to authenticate (email OTP, SSO, whatever you configure) before any
  request reaches the homelab box at all.
- **Host-level access** (SSH) is separate from the tunnel and is hardened
  independently — see step 5 below.

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
cloudflared tunnel route dns gaia gaia.yourdomain.com
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

In the Cloudflare Zero Trust dashboard, add an **Access application** for
`gaia.yourdomain.com` and set a policy for who may authenticate (your
email, a group, SSO provider, etc.). Once this is in place, nobody reaches
Ollama without passing that login — this is the "login" layer; it lives in
Cloudflare because GitHub Pages has no backend to run it.

## 5. Harden host (SSH) access, separately from the tunnel

This governs administrative access to the box itself, not the AI service.

```bash
# Disable password auth, key-only login (edit /etc/ssh/sshd_config):
#   PasswordAuthentication no
#   PermitRootLogin no
sudo systemctl restart sshd
```

Log in as a normal, non-root user with an SSH key. Root is never reached
directly — only via `sudo` from that account.

## 6. Gate root (`sudo`) behind a second factor

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
