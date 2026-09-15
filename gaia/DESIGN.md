# Gaia — design rationale

This captures *why* Gaia is built the way it is, for anyone (or any future
Claude Code session) picking this up without the original conversation.
`README.md` in this directory has the *how* (setup steps); this is the *why*.

## What this project is

A self-hosted AI assistant (Ollama) running on a personal homelab server,
reachable remotely, under the owner's full control rather than a hosted
third-party service.

## Design decisions and why

**No public, unauthenticated access to anything on the homelab.**
Early versions of this idea considered exposing homelab access directly
from the public GitHub Pages site, including with full root privileges and
a plaintext shared password. That was rejected outright: a public static
site with no auth in front of a root-capable channel is an open invitation
for anyone who finds the endpoint to fully compromise the home network —
not a hypothetical risk, but the direct, intended function of that design.
Every choice below exists to avoid that shape of failure.

**Cloudflare Tunnel instead of an exposed port.**
`cloudflared` makes an outbound-only connection from the homelab server to
Cloudflare's edge. No port is opened on the home router/firewall, so there
is no listening service on the public internet to scan, fingerprint, or
attack directly — the attack surface a port-forwarded setup would have is
absent by construction.

**Cloudflare Access in front of the tunnel.**
Authentication happens *before* any request reaches the homelab box.
GitHub Pages is static-only and cannot run a login backend, session store,
or auth logic of any kind — so the "login" requirement couldn't live on
the site itself even if we wanted it to. Access is where that requirement
is actually enforced.

**Ollama bound to `127.0.0.1` only, never `0.0.0.0`.**
Even with the tunnel and Access in place, defense in depth means the
service itself shouldn't be reachable on the LAN or any other interface —
only `cloudflared`, running on the same box, can reach it.

**SSH also routed through the tunnel, behind its own Access application.**
Administrative access needed to work from any device, not just on the
home LAN — so SSH gets a second tunnel hostname (`ssh.android21engine.org`)
with its own Cloudflare Access policy, separate from the AI's. This is
additive, not a replacement: Access gates who can even attempt an SSH
connection; the host-level hardening below still applies underneath it.
No SSH port is ever forwarded on the router — same reasoning as the
Ollama tunnel, applied to admin access instead of the AI.

**SSH: key-only login, non-root user, root gated behind a second factor.**
This is a separate privilege tier from the AI/tunnel access above — it
governs administrative control of the machine itself, independent of how
the connection arrived (tunnel or LAN). Password auth is disabled
entirely (keys only). Login lands in a normal user account, not root.
Escalating to root via `sudo` requires a TOTP code (PAM), so a leaked SSH
key alone is not sufficient to get root — an attacker would still need
the second factor. Combined with Access in front of the tunnel hostname,
reaching root now requires: passing Cloudflare Access, then an SSH key,
then a TOTP code — three independent factors.

**No credentials of any kind committed to the repo.**
Tunnel tokens, credentials JSON, and TOTP secrets are created and stored
only on the homelab machine itself (see `gaia/.gitignore`). This repo is
public (GitHub Pages) — anything committed to it is permanently public,
even if later removed from HEAD (it stays in git history). Config files
that need real secrets ship as `.example` templates only.

**Why a domain + Cloudflare Access instead of no public hostname at all.**
The alternative considered was WARP private network routing (no domain,
no public hostname, only enrolled devices can even resolve the private
IP) — arguably an even smaller attack surface. The owner chose to
register a domain and use a public hostname behind Access instead, for a
normal bookmarkable URL. This is still safe under the model above (auth
before the request ever reaches the homelab), just a deliberate trade of
"zero public surface" for convenience — worth knowing if the setup is
revisited later and someone asks "why not WARP-only?"

## Non-goals

- No training or fine-tuning of models — Ollama runs standard pre-trained
  open models as-is. "Train a new AI" was part of the original ask and is
  explicitly not what this project does.
- No attempt to strip or bypass any model's safety behavior. Self-hosting
  is about control over infrastructure and privacy, not about running an
  unrestricted model.
