# DUET Freshers' Orientation & Reception: Registration System

A free, serverless registration system for the **DUET Freshers' Orientation & Reception 2025-26**, run by Bangladesh Islami Chhatrashibir, DUET Branch.

- **Frontend:** a single static `index.html` (host it on GitHub Pages, Netlify, or any static host)
- **Backend:** one Google Apps Script (`Code.gs`) web app
- **Database:** a Google Sheet
- **Cost:** free (Google account and a free Cloudflare Turnstile widget)

---

## Features

### For students
- Registration form: Email, Full Name, Department, Student ID, Phone (optional), Comment (optional)
- Live **remaining slots** counter (on the form and on the status page)
- Instant **Digital Registration Card** with a unique Registration ID (`DUETSHIBIR-0001`, ...)
- **Print / Save PDF** and **Screenshot (PNG)** buttons, with fallbacks for iOS and in-app browsers (Facebook/Messenger)
- **Check Registration Status:** enter **Email + Student ID**. If both match one registration, the full card is shown again, with the Print and Screenshot buttons and the remaining slots
- Card also sent automatically by email (with a PDF attachment) in the background

### For organizers
- 750 slots (configurable), enforced by an atomic counter, so no overbooking
- Duplicate protection: the same Email or the same Student ID cannot register twice
- Everything is stored in one Google Sheet
- Incremental CSV backups in Google Drive (no backup emails)
- Admin helper functions (rebuild index, restore from backup, retry failed emails, card sending stats)

### Security and abuse protection
- Cloudflare **Turnstile** bot check (optional but recommended)
- `SHARED_SECRET` handshake (stops blind spam to the endpoint)
- Honeypot field
- Layered rate limits: global, per browser, per Student ID, per Email
- Strike system that temporarily blocks abusive clients
- Formula-injection protection for text written into the Sheet
- Generic error messages for rejected requests (details go only to the Executions log)
- Status check does not reveal who is registered unless **both** Email and Student ID are correct

---

## Project structure

```
.
├── index.html   # Frontend (form, status check, card, print/screenshot)
├── Code.gs      # Backend (Google Apps Script)
└── README.md
```

---

## How it works

```
Browser (index.html)
   │  GET  ?action=slots            -> remaining slots
   │  POST (action omitted)         -> register / return existing card
   │  POST {action: "card"}         -> status check (Email + Student ID -> card)
   ▼
Google Apps Script (Code.gs)
   ├─ gate_()            key check -> client id/block list -> honeypot -> global limit
   ├─ rate limits        per client / Student ID / Email (CacheService)
   ├─ Turnstile verify   Cloudflare siteverify
   ├─ O(1) index         PropertiesService (hashed Email/Student ID -> regId|row)
   ├─ slot counter       "CTR" property, reserved inside a short lock
   └─ Google Sheet       row written AFTER the lock is released
```

Key design points:

- **Fast duplicate check:** emails and Student IDs are stored as hashes in Script Properties, so a lookup is two property reads regardless of how many people registered.
- **Short lock:** only the counter and index update happen under the lock. The slow Sheet write happens after it, so many students can register at the same time.
- **Rollback:** if the Sheet write fails, the reserved slot is released.
- **Retry-safe:** if the browser retries after a network error, the server recognises the same Email + Student ID and returns the existing card instead of creating a duplicate.

---

## Google Sheet columns

| Col | Header | Notes |
|---|---|---|
| A | Timestamp | Registration time |
| B | Email | stored lowercase |
| C | Full Name | |
| D | Department | CE, EEE, ME, CSE, TE, Arch, IPE, MME, ChE, FE |
| E | Student ID | stored uppercase |
| F | Phone | optional, `01XXXXXXXXX` |
| G | Comment | optional |
| H | Reg ID | `DUETSHIBIR-0001`, ... |
| I | Card Sent | empty = pending, `Sent <time>`, or `ERROR: ...` |
| J | Last Updated | only used if update-mode is enabled |

Columns B to J are formatted as plain text by `setupSheet()` (leading zeros are kept, no formulas).

---

## Setup

### 1. Create the Google Sheet
1. Create a Google Sheet and rename the tab to **`duetshibir`** (or change `SHEET_NAME` in `Code.gs`).
2. Copy the Spreadsheet ID from the URL and put it in `SPREADSHEET_ID` in `Code.gs`.

### 2. Create the Apps Script project
1. In the Sheet: **Extensions > Apps Script** (or create a standalone project).
2. Paste the contents of `Code.gs`.
3. **Services (+) > Drive API > Add** (needed for permanent backup deletion and revision pruning).
4. Set a long random value for `SHARED_SECRET`.

### 3. (Recommended) Cloudflare Turnstile
1. Create a Turnstile widget in the Cloudflare dashboard and add your site's domain.
2. Copy the **Secret key** to Apps Script: **Project Settings > Script properties > Add** `TURNSTILE_SECRET`.
3. Copy the **Site key** to `TURNSTILE_SITE_KEY` in `index.html`.
4. Optional: set `ALLOWED_HOSTNAMES` in `Code.gs`, for example `["yourname.github.io"]`, so tokens only work from your site.

If `TURNSTILE_SECRET` is not set, the server skips the Turnstile check (the other protections still apply). If `TURNSTILE_SITE_KEY` is left as the placeholder, the widget is not shown.

### 4. Run the setup functions (once, from the Apps Script editor)
Run in this order and accept the permission prompts:

1. `setupSheet()`: prepares the Sheet, formats columns, builds the index and counters
2. `setupBackup()`: creates the 15-minute backup trigger and runs the first backup
3. `sendTestCard()`: sends a test card to your own email (check the layout)
4. `setupCardSender()`: starts automatic card emails (every 10 minutes)

### 5. Deploy the web app
1. **Deploy > New deployment > Web app**
2. **Execute as:** Me
3. **Who has access:** Anyone
4. Copy the **Web app URL**.

> After **every** change to `Code.gs`: **Deploy > Manage deployments > Edit > New version > Deploy**. Otherwise the old code keeps running.

### 6. Configure the frontend
In `index.html`, set:

```js
const API_URL = "<your web app URL>";
const API_KEY = "<same value as SHARED_SECRET in Code.gs>";
const TURNSTILE_SITE_KEY = "<your Turnstile site key>";   // or leave the placeholder to disable
```

Also tighten the Student ID pattern in **both** files so they match:

```js
// index.html
const STUDENT_ID_RE = /^[A-Za-z0-9][A-Za-z0-9\-]{3,19}$/;
// Code.gs
const STUDENT_ID_PATTERN = /^[A-Z0-9][A-Z0-9\-]{3,19}$/;
```

(If DUET IDs have a fixed format, for example 7 digits, use a strict pattern such as `/^\d{7}$/`. It is the best defence against bots that invent random IDs.)

### 7. Host the frontend
Upload `index.html` to GitHub Pages, Netlify, Cloudflare Pages, or any static host. Update the event date, time and venue in the HTML (and in `Code.gs` for the email card).

---

## Configuration reference (`Code.gs`)

| Constant | Default | Meaning |
|---|---|---|
| `SPREADSHEET_ID` | n/a | The Google Sheet to write to |
| `SHEET_NAME` | `duetshibir` | Tab name |
| `TOTAL_SLOTS` | `750` | Maximum registrations |
| `DEPARTMENTS` | 10 departments | Allowed department values |
| `STUDENT_ID_PATTERN` | `^[A-Z0-9][A-Z0-9\-]{3,19}$` | Student ID format |
| `SHARED_SECRET` | `CHANGE_ME...` | Must equal `API_KEY` in `index.html` |
| `ALLOWED_HOSTNAMES` | `[]` | Restrict Turnstile tokens to your site |
| `MAX_BODY_CHARS` | `4000` | Reject oversized requests |
| `GLOBAL_MAX_PER_MIN` | `240` | Circuit breaker for all traffic |
| `MAX_ATTEMPTS_PER_CID` | `12` | Per browser, per window |
| `MAX_ATTEMPTS_PER_ID` | `10` | Per Student ID and per Email, per window |
| `ATTEMPT_WINDOW_SEC` | `600` | Rate-limit window |
| `STRIKE_LIMIT` / `BLOCK_SEC` | `5` / `1800` | Strikes before a 30-minute block |
| `MAX_STATUS_PER_CID` / `MAX_STATUS_PER_QUERY` | `10` / `10` | Status-check limits |
| `LOCK_WAIT_MS` | `25000` | Max wait for the lock |
| `UPDATABLE_FIELDS` | `[]` | Fields a student may change by re-submitting (empty = none) |
| `BACKUP_FOLDER_NAME` | `DUET Registration Backups` | Drive folder |
| `BACKUP_KEEP_DAYS` | `14` | Older backup CSVs are deleted permanently |
| `EVENT_NAME / DATE / TIME / VENUE` | n/a | Text used on the emailed card |
| `ATTACH_PDF` | `true` | Attach a PDF to the card email |
| `CARD_QUOTA_RESERVE` | `5` | Emails kept in reserve from the daily quota |

---

## API

All requests go to the Apps Script web app URL.

### `GET ?action=slots`
```json
{ "remaining": 312 }
```

### `POST` (registration)
Sent as `text/plain` (avoids a CORS preflight that Apps Script cannot answer).

```json
{
  "email": "...", "fullName": "...", "department": "CSE",
  "studentId": "...", "phoneNumber": "01XXXXXXXXX", "comment": "...",
  "key": "<API_KEY>", "cid": "<client id>", "ts": "<turnstile token>", "hp_check": ""
}
```

Responses:
- New registration: `{ "success": true, "regId": "DUETSHIBIR-0001", "remainingSlots": 749 }`
- Already registered (same Email + Student ID): `{ "success": true, "existing": true, "regId": "...", ... }`
- Error: `{ "success": false, "message": "..." }`
- Server busy: `{ "success": false, "busy": true }` (the frontend retries automatically)

### `POST {"action": "card"}` (status check)
Request: `email`, `studentId`, plus `key`, `cid`, `ts`, `hp_check`.

Responses:
- Match: `{ "success": true, "found": true, "regId": "...", "name": "...", "department": "...", "studentId": "...", "remaining": 312 }`
- No match: `{ "success": true, "found": false }`

---

## Admin functions (run from the Apps Script editor)

| Function | Purpose |
|---|---|
| `setupSheet()` | One-time setup; safe to re-run (rebuilds the index) |
| `rebuildIndex()` | Rescan the Sheet and rebuild index and counters. **Run after any manual row edit, delete or sort**, and not while people are registering |
| `setupBackup()` / `backupNow()` | Create the backup trigger / run a backup now |
| `restoreFromBackup()` | Rebuild data from backup CSVs into a new `restored` tab (never touches the live tab) |
| `cleanupLegacyBackups()` | Delete old full-Sheet copies from the previous version |
| `setupCardSender()` / `stopCardSender()` | Start / stop automatic card emails |
| `sendTestCard()` | Send a test card to yourself |
| `retryFailedCards()` | Mark `ERROR` rows as pending again |
| `cardStatus()` | Log how many cards are sent / pending / failed and the remaining email quota |

---

## Backups

- Every 15 minutes `backupNow()` checks for changes. If nothing changed, it exits without touching the Sheet.
- `inc_YYYY-MM-DD.csv`: rows added or updated since the last run
- `full_YYYY-MM-DD.csv`: one full snapshot per day
- Files older than `BACKUP_KEEP_DAYS` are permanently deleted.
- No backup emails are sent, so the daily email quota is reserved for registration cards.

---

## Email cards

- `sendPendingCards()` runs every 10 minutes and emails every row whose **Card Sent** cell is empty, then marks it `Sent <time>`.
- It respects the daily MailApp quota (minus `CARD_QUOTA_RESERVE`) and stops before the 6-minute script limit; the next run continues.
- A failed email is marked `ERROR: ...`. Use `retryFailedCards()` to try again.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "Invalid request." | `API_KEY` in `index.html` does not match `SHARED_SECRET`, or the deployment is not updated. Check **Executions** for the real reason |
| Changes to `Code.gs` have no effect | Create a **New version** in *Manage deployments* |
| "Registration is not ready yet" | Run `setupSheet()` (the counter is missing) |
| "Verification failed or expired" | Student waited too long on the Turnstile check; complete it again. Also check the site/secret keys and the domain |
| Slots or duplicates look wrong after manual Sheet edits | Run `rebuildIndex()` |
| "Too many attempts" | Rate limit hit; wait about 10 minutes |
| Cards not emailed | Run `cardStatus()`; check the quota, authorization, and that `setupCardSender()` was run |
| Blank screenshot | The app falls back automatically; on iOS / in-app browsers press and hold the image to save, or use Print / Save PDF |
| Status check says "No registration found" | Email and Student ID must belong to the **same** registration |

---

## Notes and limitations

- `API_KEY` lives in a public web page, so it is **not a true secret**. It only stops blind spam. The real bot protection is Turnstile and the rate limits.
- Script Properties limit: 500 KB total. About 750 registrations use roughly 100 KB, which is well within the limit.
- Rate limiting uses `CacheService` (approximate, not atomic). It is fine for abuse throttling, not for exact accounting.
- Update-mode is off by default. Re-submitting with the same Email + Student ID only returns the existing card. To allow edits, list fields in `UPDATABLE_FIELDS`, for example `["fullName", "phoneNumber"]`.
- Do not hard-code real secrets (`SHARED_SECRET`, `TURNSTILE_SECRET`) in a public repository. Keep `TURNSTILE_SECRET` in Script Properties only, and use placeholders in the committed files.

---

## License / Credits

Built for Bangladesh Islami Chhatrashibir, DUET Branch.
Website: https://duetshibir.org/ · Facebook: https://www.facebook.com/bicsduet/
