# ARIA - Automotive LLM Security CTF

A self-contained "capture the flag" chatbot built for security training /
conference booths (e.g. DEF CON AI Village). Players talk to **ARIA**, the
in-vehicle assistant for a fictional car company, "Nexus Motors," and try to
recover three flags, each mapped to an OWASP LLM Top-10 style vulnerability:

| Objective | Vulnerability class          | Target to capture                                  |
|-----------|-------------------------------|-----------------------------------------------------|
| OBJ.01    | Prompt Injection (Direct & Indirect) | `flag{pr0mpt_1nj3ct10n_hij4cks_th3_wh33l}` |
| OBJ.02    | Insecure Output Handling       | `flag{1nsecur3_0utput_cr4shes_th3_d0m}`             |
| OBJ.03    | Sensitive Information Exfiltration | `sk_live_nexus_84f1c2e9b7a3d045` (a dummy leaked API key/customer record — not a `flag{...}` token, on purpose) |

Quick-action chips only cover the two safe FAQ questions (battery health,
warranty) — the injection and exfiltration paths are reachable only by
typing into the chat box, so players have to actually engage rather than
one-click their way to an answer.

**Important:** these flags are the defaults baked into this copy of the code.
Before running this at a real event, change them (see "Customizing flags"
below) so they aren't guessable from this public writeup.

All vulnerable logic and the flag values live server-side in `/api`
(Vercel Serverless Functions), never in the files served to the browser.
Players can't just "view source" to get the flags — they have to actually
exploit the bugs.

## How it's structured

```
nexus-ai-ctf/
├── index.html        # static frontend — chat UI
├── style.css          # automotive HUD styling
├── app.js             # client logic — INTENTIONALLY renders bot replies
│                       # with innerHTML instead of textContent (OBJ.02)
├── api/
│   ├── chat.js         # ARIA's "brain" — system prompt, injection
│   │                    # detection, sensitive-data filter (and its bypasses)
│   └── session.js       # sets a non-HttpOnly cookie holding FLAG2 (base64) —
│                         # the sink an XSS payload is meant to reach
├── package.json
└── vercel.json
```

Nothing here calls a real LLM API — the "chatbot" is a small rule-based
simulation, so there's no API key to manage and no per-message cost. That
keeps hosting free and the challenge deterministic and reliable for a booth
with lots of concurrent players.

## Deploying to Vercel (free tier)

### Option A — Vercel CLI (fastest)

1. Install Node.js if you don't have it (v18+).
2. Install the Vercel CLI:
   ```bash
   npm install -g vercel
   ```
3. From inside the `nexus-ai-ctf` folder:
   ```bash
   cd nexus-ai-ctf
   vercel
   ```
4. Follow the prompts:
   - "Set up and deploy?" → **Y**
   - Log in / create a free Vercel account if prompted (GitHub, GitLab, email, etc.)
   - "Link to existing project?" → **N**
   - Project name → anything, e.g. `aria-ctf`
   - Directory → `.` (current directory)
   - Override settings? → **N** (defaults are correct — no build step needed)
5. Vercel deploys and gives you a URL like `https://aria-ctf.vercel.app`.
6. For a permanent production URL (not a preview URL), run:
   ```bash
   vercel --prod
   ```

That's it — the `/api` folder is auto-detected as serverless functions, and
the root files (`index.html`, `style.css`, `app.js`) are served as static
assets. No build command, no environment variables required.

### Option B — GitHub + Vercel dashboard (no CLI)

1. Push this folder to a new GitHub repository.
2. Go to https://vercel.com → **Add New → Project**.
3. Import the GitHub repo.
4. Framework preset: **Other**. Leave build command and output directory
   blank/default.
5. Click **Deploy**.
