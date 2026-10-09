# Holol links

Static files served through the Cloudflare Worker `holol-demo-links`. This repo is the GitHub Pages origin.

| Public URL | File in this repo |
|---|---|
| https://demo.holol.app/ | `demo/index.html` (client demo) |
| https://demo.holol.app/v/ventes.mp4, `/v/camera.mp4` | `demo/v/` |
| https://crm.holol.app/<username> | `crm/<username>/index.html` (team page, PIN-locked) |

Team pages hold only ciphertext: leads + reply script are encrypted with AES-GCM, using a key derived from a 6-digit PIN (PBKDF2-SHA256, 600k iterations).
The PIN is never committed. The page asks for it once and remembers it on that phone.

## Add a team member

1. On the box, in this repo root, run:
   `python3 /workspace/holol/demo/tools/site.py <username> <leads.csv>`
   - `<username>`: lowercase a-z, 0-9, `-`
   - `leads.csv` columns: `name,city,phone,language,message`
   This creates `crm/<username>/index.html`. It also creates that member's own PIN at `/workspace/holol/<username>-pin.txt` (chmod 600) if it doesn't exist yet.
2. Commit and push to `main`.

`https://crm.holol.app/<username>` works as soon as GitHub Pages finishes building. The Worker doesn't need to change: it serves any `crm/<username>/` folder that exists and returns 404 for the rest.

To change a member's leads, run the same command again (same PIN), then commit.
To revoke a member, delete their folder and commit.

## Reply sync

Each tap on a team page (Send, Ah, La, Ma jawbch, Booked, Undo, message sends) is queued on the phone and posted to `https://crm.holol.app/api/events`. The queue works offline and retries automatically. The pill at the top shows `✓ synced` or the number of pending events.
Events are stored in the Cloudflare D1 database `holol-team-events`.

- Auth: every member has a random sync token. It lives inside that member's encrypted page, so it only becomes usable after the PIN unlock. The Worker keeps only SHA-256 hashes of the tokens, in the `SYNC_HASHES` secret: `{"members":{"<username>":"<hash>"},"admin":"<hash>","read":"<hash>"}`.
- When you add a member, `site.py` prints the hash of their new token. Add it under `members` in `SYNC_HASHES`.
- Stages: `sent`/`na`/`relance` give `contacted`, `yes` (and the video or booking messages after it) gives `replied`, `booked` gives `meeting`, and `no` gives `lost`. Undo puts the lead back to `contacted`.
- Live view: `https://crm.holol.app/admin` asks for the admin token once.
- API: `GET /api/leads` returns the current stage per lead, and `GET /api/events?after=<seq>` returns the raw events. Both accept the read token. `DELETE /api/events?id=...` needs the admin token.
