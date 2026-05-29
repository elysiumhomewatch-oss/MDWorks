# MD Works — Technical Reference
## Google Apps Script + Sheets Patterns & Hard-Won Lessons
### Morney Deetlefs · Accumulated from the Silent Auction + Tombola Project

> Paste this file at the start of any new chat that involves Google Apps Script,
> Google Sheets, or vanilla JS frontends. It contains every lesson learned the
> hard way so they never have to be learned again.

---

## The Core Stack

Every MD Works project uses this architecture unless there is a specific reason not to:

```
Static HTML/JS (Cloudflare Pages)
        ↕ fetch() REST calls
Google Apps Script Web App (doGet / doPost)
        ↕ SpreadsheetApp
Google Sheets (database)
```

**Why this works:**
- Zero monthly cost
- No server to maintain
- Google Sheets is the CMS — non-developers can update content directly
- Apps Script handles auth, validation, business logic
- Cloudflare Pages deploys from GitHub automatically on push

---

## Apps Script — Architecture Patterns

### The Dispatcher Pattern (always use this)

Never write logic directly in `doGet` / `doPost`. Use a clean action dispatcher:

```javascript
const SS = SpreadsheetApp.getActiveSpreadsheet();

function doGet(e) {
  const action = e.parameter.action || '';
  try {
    switch (action) {
      case 'getItems':   return jsonResponse(getItems());
      case 'getConfig':  return jsonResponse(getPublicConfig());
      // ... more cases
      default:           return jsonResponse({ error: 'Unknown action: ' + action });
    }
  } catch (err) {
    return jsonResponse({ error: err.message });
  }
}

function doPost(e) {
  let body = {};
  try { body = JSON.parse(e.postData.contents); } catch (_) {}
  const action = body.action || '';
  try {
    switch (action) {
      case 'register':  return jsonResponse(registerUser(body));
      // ... more cases
      default:          return jsonResponse({ error: 'Unknown action: ' + action });
    }
  } catch (err) {
    return jsonResponse({ error: err.message });
  }
}

function jsonResponse(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Why:** Clean separation, easy to add endpoints, errors always return JSON (never HTML error pages that break frontend parsing).

---

### Auth Helper Pattern

Never repeat PIN checks. Use a shared `authAdmin` helper that throws on failure:

```javascript
function authAdmin(body) {
  const pin = PropertiesService.getScriptProperties().getProperty('AdminPIN');
  if (String(body.adminPin) !== String(pin).trim()) {
    throw new Error('Unauthorized');
  }
}

// Usage in any endpoint:
function deleteItem(body) {
  authAdmin(body); // throws if wrong PIN — caught by dispatcher
  // ... rest of function
}
```

---

### Sheet Helper

Always use a helper rather than calling `getSheetByName` repeatedly:

```javascript
function getSheet(name) {
  return SS.getSheetByName(name);
}
```

---

## Apps Script — Deployment Rules (CRITICAL)

**The single most common cause of "my changes aren't working":**

Every change to `Code.gs` requires a **new deployment version**. Saving the file alone does nothing to the live Web App URL.

```
Deploy → Manage deployments → Edit (pencil) → Version: New version → Deploy
```

The live URL stays the same — only the code behind it updates.

**Checklist after any Code.gs edit:**
- [ ] Save the file
- [ ] Deploy → Manage deployments → Edit → New version → Deploy
- [ ] Hard-refresh the browser (Ctrl+Shift+R) to clear cached API responses

---

### Config vs Script Properties — When to Use Which

| Data | Where to store | Why |
|------|---------------|-----|
| Admin PIN | Script Properties | Never visible to Sheet viewers/editors |
| API keys | Script Properties | Same reason |
| Auction name, dates | Config sheet tab | Non-devs can update without touching code |
| Feature flags (e.g. TombolaActive) | Config sheet tab | Toggle without redeployment |
| Site URL, WA number | Config sheet tab | Changes don't require redeploy |

**Reading Script Properties:**
```javascript
const pin = PropertiesService.getScriptProperties().getProperty('AdminPIN');
// Returns null if key doesn't exist — always check
if (!pin) return { error: 'AdminPIN not configured' };
```

**Fallback pattern (Properties first, sheet second):**
```javascript
function getConfig(key) {
  // Check Script Properties first (secure)
  const prop = PropertiesService.getScriptProperties().getProperty(key);
  if (prop !== null) return prop;
  // Fall back to Config sheet
  const sheet = getSheet('Config');
  const data  = sheet.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === key) return data[i][1];
  }
  return null;
}
```

**Debug PIN not working:**
```javascript
function debugPin() {
  const all = PropertiesService.getScriptProperties().getProperties();
  Logger.log(JSON.stringify(all)); // {} means nothing saved
}
```
Run this in Apps Script editor → View → Logs. If it returns `{}` the property was never saved — the save button in Project Settings is easy to miss.

---

## Google Sheets — Data Reading Rules

### Rule 1: Always Cast Explicitly

Sheets cells return mixed types — number, string, boolean, Date object — depending on cell formatting and content. Never trust the raw value.

```javascript
// WRONG — will silently fail on dates or numbers stored as text
return { id: row[0], name: row[1], amount: row[4] };

// CORRECT — always cast explicitly
return {
  id:     String(row[0]),
  name:   String(row[1]),
  amount: Number(row[4]) || 0,
  active: String(row[5]).toUpperCase() === 'TRUE',
  note:   String(row[6] || '')  // handle empty cells
};
```

**Why:** JSON.stringify silently drops Date objects, turns them to null, or throws — depending on the JS engine version Apps Script uses. Explicit casting prevents all of this.

---

### Rule 2: The Date Problem (three formats)

Google Sheets returns dates in one of three formats depending on locale and how the value was entered:

| Format | Example | When it appears |
|--------|---------|-----------------|
| ISO string | `2025-11-30T20:00:00` | When you type it in as text |
| Serial number | `46025.833` | When Sheets auto-formats as Date |
| Localised string | `30/11/2025 20:00:00` | South African locale |

**Writing dates TO the sheet — always use plain string:**
```javascript
// WRONG — Date object serialises unpredictably
sheet.appendRow([new Date(), itemId, ...]);

// CORRECT — plain string, parses reliably on read
sheet.appendRow([new Date().toLocaleString(), itemId, ...]);
```

**Reading dates FROM the sheet — use parseSheetDate():**
```javascript
function parseSheetDate(val) {
  if (!val) return null;
  const str = val.toString().trim();

  // ISO string: 2025-11-30T20:00:00
  if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}/.test(str)) {
    return new Date(str);
  }

  // Sheets serial number (days since 30 Dec 1899)
  if (/^\d+(\.\d+)?$/.test(str)) {
    return new Date((parseFloat(str) - 25569) * 86400 * 1000);
  }

  // Localised: DD/MM/YYYY or YYYY/MM/DD
  if (/\d{1,4}[\/\-]\d{1,2}[\/\-]\d{2,4}/.test(str)) {
    const native = new Date(str);
    if (!isNaN(native.getTime())) return native;
    // Manual parse for DD/MM/YYYY
    const parts = str.match(/(\d+)[\/\-](\d+)[\/\-](\d+)(?:\s+(\d+):(\d+)(?::(\d+))?)?/);
    if (parts) {
      const [, a, b, c, h='0', m='0', s='0'] = parts;
      const year  = a.length === 4 ? +a : +c;
      const month = +b - 1;
      const day   = a.length === 4 ? +c : +a;
      return new Date(year, month, day, +h, +m, +s);
    }
  }

  // Last resort
  const fallback = new Date(str);
  return isNaN(fallback.getTime()) ? null : fallback;
}
```

**For Config sheet date values (TombolaActive etc):**
```javascript
// Boolean values stored as text — always check with toUpperCase()
const isActive = String(getConfig('TombolaActive')).toUpperCase() === 'TRUE';
// 'TRUE', 'true', 'True' all work. ' TRUE' (with space) will fail — .trim() if needed
```

---

### Rule 3: Row-to-Object Pattern

Always use a dedicated function to convert a sheet row array to an object.
Never index into raw row arrays in multiple places — one schema change breaks everything.

```javascript
function rowToItem(row) {
  return {
    id:           String(row[0]),
    name:         String(row[1]),
    description:  String(row[2] || ''),
    imageUrl:     String(row[3] || ''),
    startingBid:  Number(row[4]) || 0,
    currentBid:   Number(row[5]) || 0,
    status:       String(row[8] || 'open'),
    minIncrement: Number(row[9]) || 1
  };
}
// Usage: items.push(rowToItem(data[i]));
```

---

### Rule 4: The Empty Cell Problem

Empty cells return `''` (empty string) — but cells that were never written return `0`, `false`, or `''` depending on type. Always handle both:

```javascript
String(row[6] || '')  // empty cell → empty string, not 'undefined'
Number(row[4]) || 0   // empty/null → 0, not NaN
```

---

### Rule 5: getDataRange() vs getLastRow()

```javascript
// Reading all data — use getDataRange()
const data = sheet.getDataRange().getValues();
// data[0] = headers, data[1..n] = rows

// Appending — use appendRow()
sheet.appendRow([id, name, value]);

// Updating a specific cell — use getRange(row, col)
// Both row and col are 1-indexed (not 0)
sheet.getRange(rowIndex + 1, 6).setValue(newValue);
// Note: data array is 0-indexed, sheet is 1-indexed
// So data[i] corresponds to sheet row i+1
```

---

### Rule 6: LockService for Concurrent Writes

Sheets has no row locking. If two users write simultaneously (e.g. both claiming a tombola ticket) counts can go wrong. Use LockService for any write that depends on reading first:

```javascript
function claimTicket(body) {
  const lock = LockService.getScriptLock();
  try {
    lock.tryLock(8000); // wait up to 8 seconds
  } catch(e) {
    return { error: 'Server busy — please try again' };
  }
  try {
    // ... read current value, validate, write new value
    return { ok: true };
  } finally {
    lock.releaseLock(); // ALWAYS release in finally block
  }
}
```

**When to use LockService:**
- Any operation that reads a count then writes an incremented value
- Ticket claiming, bid placement at high volume
- Any "first come first served" logic

**When you don't need it:**
- Simple appends (appendRow is atomic enough for community scale)
- Read-only operations
- Config reads

---

### Rule 7: One-Time Setup Function

Every project should have a `setupSheets()` function that creates all required tabs and seeds default config. Run once after first paste.

```javascript
function setupSheets() {
  const sheets = {
    Items:   ['ID','Name','Description','Status'],
    Config:  ['Key','Value'],
    // ... all sheets
  };

  for (const [name, headers] of Object.entries(sheets)) {
    let sheet = SS.getSheetByName(name);
    if (!sheet) sheet = SS.insertSheet(name);
    if (sheet.getLastRow() === 0) sheet.appendRow(headers);
  }

  // Seed default config — check before adding to avoid duplicates
  const configSheet = getSheet('Config');
  const existing = configSheet.getDataRange().getValues().map(r => r[0]);
  const defaults = [
    ['SiteName', 'My App'],
    ['AdminPIN', '1234'], // user must change this
  ];
  defaults.forEach(([k,v]) => {
    if (!existing.includes(k)) configSheet.appendRow([k, v]);
  });

  return { ok: true, message: 'Setup complete.' };
}
```

---

## Frontend — JavaScript Patterns

### The Item Store Pattern (CRITICAL — never skip this)

**Never serialise objects into HTML `onclick` attributes.**

Names with apostrophes (`Bob's Vase`), quotes, or special characters break JSON parsing and cause `Unexpected end of input` errors that crash the entire page script.

```javascript
// WRONG — breaks on "Bob's Auction Item"
`<div onclick="openModal(${JSON.stringify(JSON.stringify(item))})">`;

// CORRECT — store object in memory, only ID goes in DOM
const itemStore = {};

function renderCard(item) {
  itemStore[item.id] = item;           // store
  return `<div onclick="openModal('${item.id}')">…</div>`; // only ID
}

function openModal(itemId) {
  const item = itemStore[itemId];      // look up
  if (!item) return;
  // ... use item safely
}
```

**Every store in the auction project:**
```javascript
const itemStore       = {};  // index.html — auction items
const prizeStore      = {};  // index.html — tombola prizes
const adminItemStore  = {};  // admin.html — auction items
const adminPrizeStore = {};  // admin.html — tombola prizes
const adminVideoStore = {};  // admin.html — sponsor videos
```

---

### The esc() Helper (always include)

Every piece of user-generated content rendered into innerHTML must be escaped.
One missing escape = XSS vulnerability + broken HTML if name contains `<` or `"`.

```javascript
function esc(str) {
  if (!str) return '';
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
// Usage: `<div>${esc(item.name)}</div>`
```

---

### The Post Helper

Single function for all POST requests to Apps Script:

```javascript
async function post(body) {
  const r = await fetch(API, {
    method: 'POST',
    body: JSON.stringify(body)
  });
  return r.json();
}
// Usage: const res = await post({ action: 'register', name, phone });
```

---

### Fetching Config on Load

Always fetch dynamic config (app name, dates, feature flags) from the sheet on page load. Never hardcode these in HTML.

```javascript
window.addEventListener('DOMContentLoaded', () => {
  fetchConfig();
  // ... other init
});

async function fetchConfig() {
  try {
    const res  = await fetch(`${API}?action=getConfig`);
    const data = await res.json();
    if (!data.ok) return;
    if (data.appName) document.title = data.appName;
    // ... apply other config
  } catch(e) {
    console.warn('Config load failed:', e);
    // fail silently — never crash the page over config
  }
}
```

---

### Active State Variable + Modal Clearing

When opening a modal from a card, save the active item to a variable.
When closing the modal, clear it — but be careful not to clear it too early if the same variable is needed by a subsequent action.

```javascript
let activePrize = null;

function openPrizeModal(prizeId) {
  activePrize = prizeStore[prizeId];
  // ... open modal
}

function closePrizeModal() {
  document.getElementById('prizeModal').classList.remove('open');
  // Only clear if no subsequent action depends on it
  if (!activeToken) activePrize = null;
}

// When closing modal AND starting an action that needs activePrize:
async function startAction() {
  const savedPrize = activePrize; // save BEFORE closing
  closePrizeModal();              // this may clear activePrize
  activePrize = savedPrize;       // restore
  // ... proceed with action
}
```

**This exact bug was hit in the tombola:** `closePrizeModal()` cleared `activePrize`, then the video completion handler tried to use `activePrize.id` and got `undefined`.

---

### Unicode Characters in JavaScript (the invisible bug)

Decorative characters like `────` (box-drawing) pasted into a JS script block cause:
```
Uncaught SyntaxError: Invalid or unexpected token
```
This crashes the **entire script** — all functions become undefined, causing confusing secondary errors like `switchTab is not defined`.

**Rules:**
- Never paste decorative box-drawing characters (`─`, `━`, `═`) inside `<script>` tags
- Comments in JS should use only standard ASCII: `// ──` is fine, `────` on its own line is not
- If you see `Invalid or unexpected token` with no obvious cause, search the file for non-ASCII characters

```bash
# To find them:
grep -n '[─━═─┌┐└┘│]' yourfile.html
```

---

### Stale CSS (the invisible layout bug)

If you see raw CSS text rendered as page content below the normal page, the `</style>` tag is missing or malformed. The browser stops parsing CSS and renders everything as text.

**Cause:** Pasting tombola CSS additions outside the existing `</style>` tag when merging files manually.

**Fix:** Always do full file replacements when provided — never manually merge CSS blocks. The `</style>` closing tag must be the only one, at the end of all styles.

---

### localStorage for Session State

Bidder ID and name persist across page refreshes via localStorage. No auth service needed.

```javascript
// On registration success:
localStorage.setItem('bidderId',   res.bidderId);
localStorage.setItem('bidderName', res.name);

// On page load:
const currentBidderId   = localStorage.getItem('bidderId')   || '';
const currentBidderName = localStorage.getItem('bidderName') || '';

// If registered, skip to main view:
if (currentBidderId) switchTab('auction');
```

**What to store:** Only non-sensitive display data. Never tokens, PINs, or payment info.

---

## Token Validation Pattern (Tombola)

For any "watch/do X then claim reward" flow, use server-side token timing:

```
1. Client requests token  → GET ?action=getToken
   Server: generate UUID token, log {token, issuedAt, status:'pending'} to sheet
   Returns: { token, issuedAt, ...other data }

2. Client does the action (watches video, completes task)

3. Client claims reward   → POST { action:'claimReward', token, userId, ... }
   Server validates:
     a. Token exists in sheet with status 'pending'
     b. elapsed >= MIN_SECONDS (action must have taken real time)
     c. elapsed <= EXPIRY_SECONDS (not stale)
     d. userId is valid
   On pass: mark token 'used', award reward, return { ok: true }
   On fail: return { error: 'descriptive message' }
```

```javascript
// Server-side validation core:
const elapsed = (new Date() - new Date(issuedAt)) / 1000;
if (elapsed < 5)             return { error: 'Action not completed' };
if (elapsed > tokenExpiry)   return { error: 'Token expired — please try again' };
if (tokenStatus !== 'pending') return { error: 'Token already used' };
```

**Key lesson:** The minimum elapsed time (5 seconds) should be well under the actual action duration. Setting it to match the action duration exactly (28s for a 30s video) fails when videos are shorter than expected. Use 5s as a fraud floor, let the action length be whatever it is.

---

## YouTube IFrame API

### The Global Hook (most common gotcha)

`onYouTubeIframeAPIReady` **must** be on `window`. YouTube's script loader calls it by name on the global object — it will silently never fire if scoped inside a module, class, or IIFE.

```javascript
// WRONG — never fires
function onYouTubeIframeAPIReady() { ... }

// CORRECT — must be global
window.onYouTubeIframeAPIReady = function() {
  // Player can be initialised here, or lazily on first use
};
```

### Player Setup with Locked Controls

```javascript
ytPlayer = new YT.Player('ytPlayer', {
  height: '100%', width: '100%',
  videoId: videoId,
  playerVars: {
    autoplay:        1,
    controls:        0,  // hide all controls
    disablekb:       1,  // disable keyboard shortcuts
    fs:              0,  // no fullscreen button
    modestbranding:  1,  // minimal YouTube branding
    rel:             0   // no related videos at end
  },
  events: {
    onStateChange: onPlayerStateChange,
    onReady: e => e.target.playVideo()
  }
});
```

### Completion Detection

```javascript
function onPlayerStateChange(event) {
  if (event.data === YT.PlayerState.ENDED && !videoCompleted) {
    videoCompleted = true; // prevent double-firing
    claimReward();
  }
}
```

### Dynamic Countdown from Actual Duration

Don't hardcode a countdown — read the actual video duration from the player:

```javascript
videoTimerHandle = setInterval(() => {
  if (!ytPlayer || typeof ytPlayer.getDuration !== 'function') return;
  const duration  = Math.ceil(ytPlayer.getDuration()) || 30;
  const current   = ytPlayer.getCurrentTime() || 0;
  const remaining = Math.max(0, Math.ceil(duration - current));
  const m = Math.floor(remaining / 60);
  const s = remaining % 60;
  document.getElementById('timer').textContent = `${m}:${String(s).padStart(2,'0')}`;
  if (remaining <= 0) clearInterval(videoTimerHandle);
}, 1000);
```

### postMessage Errors

YouTube's IFrame sends `postMessage` calls internally. You will see console errors like:
```
Failed to execute 'postMessage' on 'DOMWindow': The target origin provided does not match
```
These are **harmless** — YouTube talking to itself across iframe boundaries. Ignore completely.

---

## ImgBB Image Upload

```javascript
const IMGBB_KEY = 'YOUR_IMGBB_API_KEY'; // free at imgbb.com

async function uploadImage(file) {
  const formData = new FormData();
  formData.append('image', file);
  formData.append('key', IMGBB_KEY);

  const res  = await fetch('https://api.imgbb.com/1/upload', {
    method: 'POST', body: formData
  });
  const data = await res.json();

  if (data.success) {
    return data.data.url; // direct image URL — store in sheet as plain string
  } else {
    throw new Error(data.error?.message || 'Upload failed');
  }
}
```

**File size limit:** 32MB per image. Check before uploading:
```javascript
if (file.size > 32 * 1024 * 1024) {
  showToast('Image must be under 32MB', 'error');
  return;
}
```

---

## WhatsApp Integration

### wa.me Deep Link Format

```javascript
const message = `*Bold text*\n_Italic text_\n----\nPlain text`;
const href = `https://wa.me/?text=${encodeURIComponent(message)}`;
window.open(href, '_blank');

// To open to a specific number:
const href = `https://wa.me/27831234567?text=${encodeURIComponent(message)}`;

// Number format: country code + number, digits only
// South Africa: 27 + 9 digits (no leading zero)
// 083 123 4567 → 27831234567
```

### The Emoji Rule

**Never use emoji in wa.me messages.** They render correctly on Android but display as `◆?` (black diamond with question mark) on Windows Chrome when URL-encoded.

```javascript
// WRONG
`🥇 *${item.name}*\n💰 Current bid: R ${amount}`

// CORRECT
`1. *${item.name}*\nCurrent bid: R ${amount}`
```

**Safe WA formatting (works on all platforms):**
- `*text*` → **bold**
- `_text_` → _italic_
- `----` → horizontal divider
- Plain text, numbers, line breaks — always safe

### Number Stripping for Winner Notifications

Phone numbers from registration may have spaces, dashes or leading zeros:

```javascript
const clean = phone.replace(/\D/g, ''); // strip all non-digits
const href  = `https://wa.me/${clean}?text=${encodeURIComponent(msg)}`;
```

---

## Apps Script — Free Tier Limits

Relevant limits for community-scale projects:

| Limit | Value | Notes |
|-------|-------|-------|
| Script execution time | 6 minutes | More than enough per request |
| Daily URL fetch calls | 20,000 | For external API calls from Script |
| Spreadsheet read/write | No hard limit | Throttled at very high volume |
| Web App requests | No hard limit | Google throttles abuse |
| Script Properties | 500KB total | More than enough for config |
| Email sends (MailApp) | 100/day (free) | If adding email features |

For community events (hundreds of users, not thousands) none of these limits are a concern.

---

## Cloudflare Pages Deployment

```
GitHub repo push → Cloudflare auto-deploys in ~30 seconds
No build command — pure static files
```

**Editing files:**
1. Edit locally in VS Code
2. Push to GitHub
3. Cloudflare auto-deploys

**Or:** Cloudflare Pages → Deployments → Create new deployment → upload files directly

**After updating HTML files:** Hard-refresh browser (`Ctrl+Shift+R`) — Cloudflare caches aggressively.

**After updating Code.gs:** Must redeploy Apps Script as a new version (Cloudflare not involved — this is on the Google side).

---

## Debugging Reference

### "My changes aren't working"
→ Did you redeploy Apps Script as a new version? 95% of the time this is the cause.

### "Incorrect PIN"
→ Run `debugPin()` in Apps Script. If it returns `{}` the property was never saved.
→ Check for spaces in the PIN value in Script Properties.

### "Error loading [data]"
→ Paste `YOUR_API_URL?action=getWhatever` directly in the browser.
→ Read the raw JSON response — it will contain a specific error message.

### "Connection error" after video
→ Check `activePrize` is not null. The modal close may have cleared it before the video ended.
→ Add `console.log('Claiming with:', { token, bidderId, prizeId })` before the fetch.

### "Unexpected end of input" + secondary errors
→ Search the JS script block for non-ASCII characters (box-drawing, decorative dashes).
→ The entire script fails to parse — all functions appear undefined.

### "Raw CSS text visible below page"
→ The `</style>` tag is missing or appears before all the CSS.
→ Do a full file replacement, never manually merge CSS additions.

### "Winners sheet date shows as number"
→ The cell in Sheets is formatted as Date (shows serial number).
→ Fix: format the column as Plain Text. The platform handles parsing automatically.

### "TombolaActive is FALSE even though I set it to TRUE"
→ Check the exact string. Must be `TRUE` — not `True`, not ` TRUE` (space).
→ `String(val).toUpperCase() === 'TRUE'` handles case but not spaces.

---

## Session Opener for New Chats

Paste this + attach this file at the start of any session involving Apps Script or Sheets:

> *"I'm Morney Deetlefs (MD Works). I build zero-cost community tools using vanilla JS, Google Apps Script and Google Sheets. All hard-won patterns from previous projects are in the attached md-works-technical.md — please follow them. Currency is ZAR (R). WA messages use no emoji."*

