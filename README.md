# Holol links

Static files served through the Cloudflare Worker `holol-demo-links`. This repo is the GitHub Pages origin.

| Public URL | File in this repo |
|---|---|
| https://demo.holol.app/ | `demo/index.html` (client demo) |
| https://demo.holol.app/v/ventes.mp4, `/v/camera.mp4`, `/v/ventes-hd.mp4`, `/v/camera-hd.mp4` (full quality) | `demo/v/` |
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

### Message style (first messages and reply script)

Darija is written in Arabic script, French words stay in Latin where people naturally mix them. Write in Chaimae's own voice:
her name and that she is from Kenitra (even for leads in other cities), "I saw your place", what it does for them (less tamara, the day's money clear, grow the place), then the video offer: "إلا بغيتي تفهم كثر نصيفط ليك فيديو لي يشرح ليك كلشي".
No dashes, no tech words. The reply script lives in `site.py`.

### Message variants (A/B/C test)

First messages come in 3 variants, defined in `/workspace/holol/leads/variants.py` (personalized with place name, city and category):
A = know the day's money without being there, B = less tamara with cash and staff, C = save time and grow the place.
The leads CSV has `category` and `variant` columns; when `variant` is set, the CSV `message` is sent as is. Variants are assigned round robin over leads sorted by city and category, so each one gets a similar mix.
Every event from a team page carries the lead's variant, so `https://crm.holol.app/admin` shows a "Message variants" table: Sent (leads sent), Replied (answered آه or لا), Reply rate, Booked.

How to read it: wait for about 10 sends per variant before judging. Reply rate is the main signal, Booked breaks ties. Leads sent before variants existed have no variant and are not counted.
To retire a losing variant: in the CSV, set the unsent leads that have that variant to one of the remaining variants and rebuild their `message` with `variants.message(...)`, rerun `site.py` (same PIN) and push. Leads already sent keep their variant so past results stay correct. To try a new idea, add a new letter to `T` in `variants.py` and give it to some unsent leads the same way.

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
