# mParivahan — Vercel + GitHub Setup Guide

## What changed from the old version
| Old (Render)             | New (Vercel + GitHub)                         |
|--------------------------|-----------------------------------------------|
| APK stored in Telegram   | APK pushed to GitHub repo automatically        |
| Polling mode             | Webhook mode (required for Vercel serverless)  |
| store.json on disk       | In-memory + GitHub as source of truth          |
| No /rename               | /rename command to rename the download file    |
| Filename truncation bug  | Fixed — was stripping valid chars from name    |

---

## Step 1 — Create the GitHub APK repo

1. Go to https://github.com/new
2. Name it `mpari-apk` (or anything — put it in `GITHUB_REPO` env var)
3. Make it **Public** (so the APK raw URL is downloadable without auth)
4. Initialize with a README so it's not empty
5. Create the folder `releases/` — easiest: add a dummy file `releases/.gitkeep`

---

## Step 2 — Generate a GitHub Personal Access Token

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Click "Generate new token"
3. Resource owner: your account
4. Repository access: Only select repositories → pick `mpari-apk`
5. Permissions → Repository permissions:
   - Contents: **Read and write**
6. Generate & copy the token → this goes in `GITHUB_TOKEN`

---

## Step 3 — Deploy to Vercel

```bash
npm i -g vercel
vercel login
cd mpari-vercel
vercel --prod
```

Or: Push to GitHub and import at https://vercel.com/new

---

## Step 4 — Set environment variables on Vercel

Go to your project → Settings → Environment Variables, add:

| Variable         | Value                                  |
|------------------|----------------------------------------|
| `BOT_TOKEN`      | Your Telegram bot token                |
| `ADMIN_ID`       | Your Telegram chat ID (get via @userinfobot) |
| `GITHUB_TOKEN`   | The fine-grained token from Step 2     |
| `GITHUB_OWNER`   | Your GitHub username                   |
| `GITHUB_REPO`    | `mpari-apk` (or whatever you named it) |
| `GITHUB_BRANCH`  | `main`                                 |
| `VERCEL`         | `1`                                    |
| `WEBHOOK_SECRET` | Any random string (e.g. `abc123xyz`)   |

---

## Step 5 — Register the Telegram Webhook

After deploying, run this **once** in your browser or curl:

```
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-vercel-domain>/api/webhook&secret_token=<WEBHOOK_SECRET>
```

Replace:
- `<BOT_TOKEN>` — your bot token
- `<your-vercel-domain>` — e.g. `mpari.vercel.app`
- `<WEBHOOK_SECRET>` — the same value you set in env vars

Verify it worked:
```
https://api.telegram.org/bot<BOT_TOKEN>/getWebhookInfo
```

---

## Step 6 — Update index.html download URL

In `index.html`, wherever the APK download button calls `/get-apk`, update to `/api/get-apk`:

```js
// OLD
window.location.href = '/get-apk';

// NEW
window.location.href = '/api/get-apk';
```

---

## Bot commands

| Command                          | Who       | What it does                      |
|----------------------------------|-----------|-----------------------------------|
| `/start` or `/help`             | All       | Show available commands            |
| `/status`                        | All auth  | Show current APK info + GitHub link|
| `/rename <new_name.apk>`        | All auth  | Rename what users download as      |
| `/delete`                        | Admin     | Clear current APK from store       |
| `/addworker <chat_id> <name>`   | Admin     | Grant upload access                |
| `/removeworker <chat_id>`       | Admin     | Revoke upload access               |
| `/workers`                       | Admin     | List all workers                   |
| `/myaccess`                      | Worker    | Check own access status            |
| Send .apk file                   | All auth  | Upload APK → GitHub → site live    |

---

## Local development

```bash
npm install
cp .env.example .env
# edit .env — set USE_WEBHOOK=0 for polling locally (comment out VERCEL=1)
npm run dev
```

---

## How the APK name bug was fixed

**Old code (broken):**
```js
const safeName = String(d.current.name).replace(/["\\r\\n]/g, '');
//                                                    ^^^ ^^
// In a JS string literal, \\r = literal backslash+r, not carriage return
// This regex was matching 'r' and 'n' characters — cropping filenames!
```

**New code (fixed):**
```js
const safeName = rawName.replace(/[\r\n"]/g, '');
// Correct regex inside a character class: actual CR, LF, and double-quote
```

That's why `mparivhan.apk` showed up as `mparivha.apk` — the `n` was being stripped.
