# LubeLogger Stack (Synology NAS)

[LubeLogger](https://github.com/hargata/lubelog) vehicle maintenance and fuel
mileage tracker, running on the Synology (synology920, `192.168.1.90`) via
Komodo, with Pocket ID as an OIDC login option.

| Service | Port | URL |
|---|---|---|
| LubeLogger | 8080 | https://lube.apps.hematic.net |

Rename the hostname before first deploy if you'd rather not use `lube` — it
appears in this compose (`OpenIDConfig__RedirectURL`), the Pocket ID client's
callback URL, Traefik, and Cloudflare DNS, so pick once.

## Data

| Mount | Contents |
|---|---|
| `/volume1/containers/lubelogger/data` | SQLite database, uploaded receipts/photos |
| `/volume1/containers/lubelogger/keys` | ASP.NET Data Protection keys — **required**, without persistence every restart invalidates existing login sessions/cookies |

Both are local NAS disk (this stack runs *on* the Synology, so `/volume1/containers`
here is local I/O, not a remote NFS mount — no conflict with the
"databases never on NFS" rule, which is about a *different* host reaching
storage over the network).

## Environment (`stack.env`)

| Variable | Description |
|---|---|
| `TZ` | Timezone |
| `LUBELOGGER_OIDC_CLIENT_ID` | From the Pocket ID "LubeLogger" client |
| `LUBELOGGER_OIDC_CLIENT_SECRET` | From the same client |

## Traefik

`lube.apps.hematic.net` → `http://192.168.1.90:8080` in
`traefik_stack/config/dynamic/combined-services.yml` (middlewares:
`default-headers, geoblock` — no forwardAuth; LubeLogger guards itself via
its own login + optional OIDC, same shape as paperless/komodo/komga).

## Setup

1. **Deploy** this stack via Komodo on synology920. Leave the two OIDC env
   vars as placeholders for now (`OpenIDConfig__ClientId`/`Secret` blank is
   fine — LubeLogger just won't show the OIDC button yet).
2. Visit `https://lube.apps.hematic.net`, complete first-run setup, create
   your account. **Note the email you use** — OIDC matches by email, so it
   must equal your Pocket ID user's email.
3. **Pocket ID** (`id.apps.hematic.net`) → OIDC Clients → Add:
   - Name: `LubeLogger`, launch URL `https://lube.apps.hematic.net`
   - Callback URLs (both — the second is LubeLogger's own diagnostic page,
     handy for a one-shot troubleshooting check if the first attempt fails):
     - `https://lube.apps.hematic.net/Login/RemoteAuth`
     - `https://lube.apps.hematic.net/Login/RemoteAuthDebug`
   - Public **off**, PKCE **off** (matches `UsePKCE=false` above), Skip
     Consent **on**
   - **Save**, then **grant a user group** on the client (Pocket ID v2
     restricts every new client by default — no grant means an instant
     `access_denied` with no login prompt, the same trap hit on paperless/
     komodo/audiobookshelf). `household` if this is a shared family log,
     otherwise just your admin group.
   - **Tick "Email verified"** on your Pocket ID user if not already done —
     required, same as the Komga integration.
   - Copy the Client ID and secret.
4. Put those two values into the Komodo stack env, **redeploy** (env vars
   need a container recreate to take effect).
5. Log out, use the OIDC button → passkey → back into your account (matched
   by email).

## Notes

- `OpenIDConfig__DisableRegistration=true`: OIDC logins only work for
  **existing** LubeLogger accounts (created in step 2) — nobody gets a new
  account auto-created from an OIDC login. Set to `false` only if you want
  that.
- `OpenIDConfig__DisableRegularLogin` stays `false` during the trial so the
  password form keeps working as a fallback. Flip to `true` later if you
  want OIDC-only.
- LubeLogger supports exactly one OIDC provider at a time — fine here since
  Pocket ID is the only IdP in this homelab.
- Multi-vehicle/multi-user: if this becomes a shared household log, repeat
  step 2 (one LubeLogger account per person) before granting them Pocket ID
  access — the email-match requirement means each person needs both sides
  created first.
