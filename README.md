# Synology NoteStation MCP Server (2FA)

Model Context Protocol (MCP) server for **Synology Note Station** — lets an AI
assistant read and write notes on a NAS via the DSM backend API, **including
on DSM accounts with two-factor auth enabled**.

This is a fork of
[banffyang/synology-notestation-mcp](https://github.com/banffyang/synology-notestation-mcp),
patched for DSM two-factor auth and for safe (recoverable) note deletion.

Verified 2026-10-17 against: Note Station 2.6 build 3938, DSM 7 with 2FA
enabled (admin account).

## What this fork patches (vs upstream)

1. **DSM 2FA login** (`src/dsm-api.ts`) — first auth call returns a 403
   challenge; the client computes TOTP locally from `SYNO_OTP_SECRET` (base32)
   and retries with `otp_code` + `otp_token` (+ `otp_ticket`). The challenge
   token appears under `data.token` or `error.errors.token` depending on DSM
   version — both shapes handled. Fallbacks: `SYNO_OTP_FILE` or a static
   `SYNO_OTP_CODE`. Restart = automatic login, no manual steps.
2. **Update uses method `set`, not `update`** (`src/notestation.ts`) —
   upstream's `update` fails with DSM error 103.
3. **`rawApiCall` param-clobber fix** (`src/dsm-api.ts`) — extra params no
   longer overwrite `api`/`version`/`method`/`_sid`.
4. **Safe delete + restore** (`src/notestation.ts`, `src/index.ts`) — see
   "Delete safety" below.

## Setting up 2FA in your environment

The server performs DSM's full 2FA login flow **automatically at startup** —
no manual code entry, ever. You only need to tell it where to get the code
from, one time:

1. **Prerequisite**: 2FA must be enabled on your DSM account
   (DSM → Control Panel → Security → 2FA), and the second sign-in step must
   include **Verification code (OTP)** — the 6-digit-code method. This
   server logs in by computing that code itself, so the code method is
   required: an account that only uses "Approve sign-in" (push via the
   Synology Secure SignIn app) or a hardware key cannot be logged into via
   the API. If you don't use 2FA, skip this section entirely — no OTP env
   var is needed and the server works as-is.
2. **Get your base32 TOTP secret** (`SYNO_OTP_SECRET`): this is the secret
   behind the 6-digit codes your authenticator app generates for DSM.
   In order of preference:
   - **During DSM 2FA setup** (best): in the 2FA wizard, click
     **"Can't scan it?"** next to the QR code — DSM reveals the secret key
     as plain text (per Synology's official DSM 7 help). The secret is only
     shown in this wizard, so grab it then.
   - **From your authenticator app** (already set up): some apps expose the
     key for an entry — Google Authenticator **on Android**: tap the
     Synology account, the details view shows the secret; **Aegis**: shows
     the secret per entry. (Google Authenticator on iOS does not.)
   - **Export from Google Authenticator (Android)**: Settings → *Transfer
     accounts* → *Export existing accounts* — the export QR encodes an
     `otpauth://totp/...?secret=<BASE32>` URI; decode it with any QR reader
     and copy the `secret=` value.
   - **Reset the OTP device in DSM** (re-runs the setup wizard, which shows
     a fresh QR + the "Can't scan it?" secret link): **Personal** (user icon,
     top right) → **Security** tab → *2-Factor Authentication* →
     **Manage Devices for 2-Factor Authentication** → **Verification code
     (OTP)** tab → **Reset Devices** (under "Change Mobile Device") →
     confirm with your DSM password. (Your old app entry stops working for
     DSM after this.) Verified against DSM 7.2 UI, 2026-10-17.
   - Not recoverable from the NAS itself: DSM stores it encrypted
     (`/usr/syno/etc/packages/SecureSignIn/preference/<user>/`).
   - Keep the secret somewhere safe — it's equivalent to a password.
3. **Add the env vars to your MCP client's config** so the server process
   receives them at launch. OpenCode example (project `opencode.json`;
   adjust the path to your clone):

   ```json
   {
     "mcp": {
       "notestation": {
         "type": "local",
         "command": ["node", "/absolute/path/to/synology-notestation-mcp-2fa/dist/index.js"],
         "enabled": true,
         "environment": {
           "SYNO_HOST": "192.0.2.10",
           "SYNO_PORT": "5000",
           "SYNO_HTTPS": "false",
           "SYNO_USERNAME": "your-dsm-user",
           "SYNO_PASSWORD": "your-dsm-password",
           "SYNO_OTP_SECRET": "BASE32-SECRET-FROM-STEP-2"
         }
       }
     }
   }
   ```

   Other MCP hosts use their equivalent `env`/`environment` block for local
   (stdio) servers. See the [Configuration table](#configuration) for all
   vars and defaults. **Never commit the file holding these values.**
4. **Restart the MCP host.** On startup the server logs in, receives DSM's
   2FA challenge, computes the current TOTP from the secret, and completes
   the login — every restart and every session works with zero manual steps.

**Can't find the TOTP secret?** Two fallbacks (in order of preference):
- `SYNO_OTP_FILE` — path to a small file containing the *current* 6-digit
  code. Good if you can export the secret to a script or prefer a hardware
  token; the file is re-read on every login, so update it with a fresh code
  when needed.
- `SYNO_OTP_CODE` — a static code passed directly. Only works for the ~30 s
  window the code is valid; useful for one-off testing, not for daily use.

## Delete safety (never regress)

- DSM `SYNO.NoteStation.Note` `delete` **without** `recycle` is a
  **PERMANENT delete** — it bypasses the Recycle Bin entirely. (Upstream's
  `delete_note` omitted the param, so it silently hard-deleted.)
- The web UI always passes `recycle`: normal delete → `recycle=true` (bin);
  empty-bin → `recycle=false` (hard delete).
- This fork: `delete_note` **defaults to the Recycle Bin**
  (`recycle: "true"`); `permanent: true` hard-deletes (use only on explicit
  user request). `set` also accepts a `permanent`-style opt-in only — never
  change the default.
- `restore_note` (method `restore`, param `object_id`) brings a bin note back
  to its original notebook. Verified round-trip: delete→bin, restore→notebook,
  re-delete→bin.
- Bin notes remain visible in `list` with `recycle: true` and
  `parent_id: "1034_#00000000"` (the bin "notebook"), so nothing vanishes
  silently. Note: renaming a bin note via `set` fails (1049) — leave bin-note
  titles as-is.

## DSM API quirks (observed, verified 2026-10-17)

| Quirk | Detail |
|---|---|
| `set` silently ignores unknown params | A "success" response does **not** mean the field changed — always re-verify via `list`/`get` (e.g. `set recycle=true` is ignored). |
| Tags must be a JSON-array string | `tags: ["a","b"]` works; comma-separated → error 114 (no permission). |
| `update` method doesn't exist | Use `set`. |
| Error 103 is ambiguous | Returned for unknown method *and* unknown param — even a fake method name (`zzz_not_real`) returns 103, so it can't distinguish the two. |
| Error 114 | No permission (e.g. malformed tags). |
| Error 403 | 2FA challenge (handled in login flow). |
| TOTP code is single-use per window | DSM consumes a 6-digit code after ONE login; a second login in the same 30 s window (e.g. MCP server restart) gets a bare **404** on the OTP retry (after ~6 s). Login handles this: on a 404 it waits for the next 30 s window and retries with a fresh code. |
| Error 1033 w/ `synodrive:1008` | Transient `delete` failure — retry works (seen repeatedly; retry logic is fine). |
| `SYNO.API.Info` | `query=all` works but returns no `methods`; `methodquery` variants → 103. API discovery via `query=all` + filter on `SYNO.NoteStation.*` is the reliable path. |
| Versions | Client uses `maxVersion` from discovery (Note = v4); the UI uses v3 for `delete`, but v4 also honors `recycle`. |
| Note id | `object_id` (`1034_...`); create takes `notebook_id` (maps to `parent_id`). Create is occasionally flaky — `raw_api_call` as fallback. |

## When the API is ambiguous: read the UI's JS bundle

The fastest way to find the exact call the Note Station UI makes is to grep
the shipped bundle on the NAS:

```bash
# <user>@<nas-host>: your SSH access to the NAS (any account that can read the file)
ssh <user>@<nas-host> 'grep -o -E ".{120}<keyword>.{120}" /var/packages/NoteStation/target/ui/notestation.js'
```

This is how the `recycle` param and the `restore` method were found — the API
probes gave no signal (103 for everything), but the UI's storage layer
contains every webapi call with its full param list
(`Ext.copyTo({},t,"object_id,recycle,perm_from,smart_id")`). Prefer this over
brute-force param guessing.

## Configuration

All configuration is via environment variables — the source contains no
hardcoded hosts, credentials, or paths. Set them in your MCP client's config
(OpenCode: the `env` block under `mcp.<server>` in `opencode.json`; other
MCP hosts: their equivalent). The file holding these values contains your
plaintext DSM password and TOTP secret — keep it out of version control.

| Var | Purpose | Default |
|---|---|---|
| `SYNO_HOST` | NAS hostname or IP (**required**) | — |
| `SYNO_PORT` | DSM port | `5001` |
| `SYNO_HTTPS` | Use HTTPS | `true` |
| `SYNO_IGNORE_CERT` | Ignore self-signed cert errors | `false` |
| `SYNO_USERNAME` | DSM account (**required**) | — |
| `SYNO_PASSWORD` | DSM password (**required**, or OS credential store — see below) | — |
| `SYNO_OTP_SECRET` | Base32 TOTP secret — current 2FA code computed locally at login (primary 2FA pathway; or OS credential store — see below) | — |
| `SYNO_OTP_FILE` | Path to a file containing the current 6-digit code (2FA fallback) | — |
| `SYNO_OTP_CODE` | Static 2FA code (fragile fallback — a code expires after 30 s) | — |

2FA is only attempted when DSM actually challenges the login (error 403);
on accounts without 2FA no OTP env var is needed.

### OS credential store (optional — no plaintext in the config file)

If you'd rather not keep the password or TOTP secret in your config file,
store them in the OS credential store (Windows Credential Manager, macOS
Keychain, or Linux Secret Service) and omit `SYNO_PASSWORD` /
`SYNO_OTP_SECRET` from the config. When a config value is absent, the server
reads the credential store via the optional `keytar` dependency (account =
your `SYNO_USERNAME`).

One-time storage (run from the repo folder; `keytar` is installed by
`npm install`):

```
node -e "require('keytar').setPassword('synology-dsm','<username>','<password>')"
node -e "require('keytar').setPassword('synology-dsm-otp','<username>','<base32-secret>')"
```

Service names: `synology-dsm` (password), `synology-dsm-otp` (TOTP secret).
Remove with `deletePassword` the same way. Config values take precedence
when both are set. On Windows the store is DPAPI-encrypted (tied to your
login session) and is not inherited by or visible to other processes.

## Build & run

```bash
npm install
npm run build     # tsc -> dist/
npm start         # stdio MCP server
```

**Operational note:** the running OpenCode MCP process loads `dist/index.js`
at startup — source edits only take effect after `npm run build` **and** an
OpenCode restart. Until a restart, use the `raw_api_call` tool for anything
the running build doesn't support (e.g. `delete` with `recycle: "true"`,
method `set`).

## MCP tools (13)

| Tool | Description |
|---|---|
| `list_notebooks` | List notebooks with note counts |
| `list_notes` | List notes, pagination; includes bin notes (`recycle: true`) |
| `get_note` | Full note content |
| `create_note` | Create note (`notebook_id` → `parent_id`) |
| `update_note` | Update via method `set` |
| `delete_note` | **Bin by default**; `permanent: true` hard-deletes |
| `restore_note` | Restore a bin note to its original notebook |
| `list_tags` / `list_todos` / `list_smart_notes` | Reads |
| `get_info` / `get_settings` | Note Station info/settings |
| `raw_api_call` | Direct DSM API call — the live exploration tool |

## Project structure

```
src/
├── dsm-api.ts          # DSM auth (incl. 2FA/TOTP), session, discovery, fetch
├── notestation.ts      # NoteStation CRUD wrapper (set/restore/safe-delete)
└── index.ts            # MCP server entry point (13 tools)
dist/                   # built output (not committed; rebuild after changes)
test-client.mjs         # stdio smoke test (upstream); reads the same SYNO_* env vars, nothing hardcoded
```

## License

MIT (inherited from upstream).
