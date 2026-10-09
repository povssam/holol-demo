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
