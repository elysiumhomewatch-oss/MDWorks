# Silent Auction Platform — Project Brief

## Overview
A zero-cost silent auction web platform for small community and charity events.
Two static HTML pages + Google Apps Script REST API + Google Sheets database.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vanilla HTML, CSS, JavaScript — no frameworks, no build step |
| Backend API | Google Apps Script Web App (doGet / doPost dispatcher) |
| Database | Google Sheets (5 tabs) |
| Image hosting | ImgBB free API (drag & drop upload in admin) |
| Notifications | WhatsApp wa.me deep links |
| Hosting | GitHub Pages / Netlify Drop (static files) |
| Auth | Script Properties for admin PIN · localStorage for bidder session |
| Cost | R0 / $0 — fully free tier |

---

## Files

| File | Purpose |
|------|---------|
| `Code.gs` | All backend logic — paste into Google Apps Script |
| `index.html` | Public page — register + browse items + place bids |
| `admin.html` | Admin dashboard — PIN login, manage items, winners, WA sharing |

---

## Google Sheets Structure (5 tabs)

| Sheet | Key Columns |
|-------|-------------|
| Items | ID, Name, Description, Image URL, Starting Bid, Current Bid, Highest Bidder ID, Highest Bidder Name, Status, Min Increment |
| Bids | Timestamp, Item ID, Item Name, Bidder ID, Bidder Name, Amount |
| Bidders | Bidder ID, Name, Phone, Email, Registered At |
| Config | Key / Value pairs (AuctionName, AuctionEnd, WhatsAppNum, SiteURL) |
| Winners | Item ID, Item Name, Winner ID, Winner Name, Winner Phone, Winning Bid, Timestamp |

---

## Apps Script Endpoints

| Method | Action | Description |
|--------|--------|-------------|
| GET | `getItems` | All items with current bid state |
| GET | `getItem&id=X` | Single item |
| GET | `getConfig` | AuctionName, AuctionEnd, WhatsAppNum, SiteURL |
| GET | `getTop3` | Top 3 open items sorted by bid count |
| GET | `getWinners` | Winners list from Winners sheet |
| POST | `register` | Register new bidder, returns BidderID |
| POST | `placeBid` | Validate and record a bid |
| POST | `adminLogin` | Verify admin PIN against Script Properties |
| POST | `addItem` | Add new auction item |
| POST | `updateItem` | Edit existing item |
| POST | `closeItem` | Close bidding on one item |
| POST | `deleteItem` | Delete an item |
| POST | `closeAll` | Close all items + write Winners sheet |
| POST | `setup` | One-time sheet creation with headers |

---

## Key Solutions That Worked

### Secure Admin PIN
Stored in **Apps Script Script Properties** (not in the sheet).
Read via `PropertiesService.getScriptProperties().getProperty('AdminPIN')`.
Never visible to anyone with Sheet access.

### Item Store Pattern (critical)
Never serialise item objects into HTML `onclick` attributes — apostrophes and quotes in item names break JSON parsing and cause `Unexpected end of input` errors.
**Solution:** store items in a JS object keyed by ID (`itemStore[item.id] = item`), pass only the plain ID string into the DOM, look up on click.
```javascript
const itemStore = {};
// In itemCard():   itemStore[item.id] = item;
// In onclick:      onclick="openBidModal('${item.id}')"
// In handler:      activeBidItem = itemStore[itemId];
```
Used in both `index.html` (`itemStore`) and `admin.html` (`adminItemStore`).

### Google Sheets Date Parsing
Sheets returns dates in three formats depending on locale and input method:
- ISO string: `2025-11-30T20:00:00`
- Serial number: `46025.833` (days since 30 Dec 1899)
- Localised string: `30/11/2025 20:00:00`

**Solution:** `parseSheetDate(val)` function handles all three cases with regex detection.
Dates written back to the sheet use `.toLocaleString()` (plain string) to avoid re-serialisation issues.

### WhatsApp Emoji Compatibility
Emoji render perfectly on Android but show as black diamonds (`◆?`) on Windows Chrome when URL-encoded in wa.me links.
**Solution:** drop all emoji, use WhatsApp's own markdown instead:
- `*text*` for bold
- `_text_` for italic
- `----` for dividers
These are processed by WhatsApp after paste, not by the browser, so they work universally.

### Currency
Defined as a single constant at the top of each script:
```javascript
const CURRENCY = 'R ';
```
Used as `${CURRENCY}${amount.toFixed(2)}` throughout — one place to change.

### Config as Single Source of Truth
AuctionName, AuctionEnd, WhatsAppNum and SiteURL all live in the Config sheet tab.
Frontend fetches via `?action=getConfig` on load — no hardcoded strings in HTML.
Admin dashboard reads the same endpoint on login to populate the topbar.

### Image Uploads
ImgBB free API — admin drag-drops an image, it uploads to ImgBB, URL auto-fills the hidden input, saves to Sheets as a plain URL string. Paste-URL fallback also available.
```javascript
const IMGBB_KEY = 'YOUR_IMGBB_API_KEY';
// POST to https://api.imgbb.com/1/upload with FormData
```

### All Data Returned as Strings
Google Sheets cells can return numbers, dates, booleans or strings unpredictably.
**Solution:** wrap all sheet reads in explicit type casts:
```javascript
String(row[0]), Number(row[4]) || 0
```
Prevents silent JSON serialisation failures.

---

## Design System

### Public Page (`index.html`)
- Dark luxury aesthetic — near-black background (`#100e0a`), gold accents (`#c9a84c`)
- Fonts: Cinzel (headings), Cormorant Garamond (display), Raleway (body)
- Tab-switch between Register and Auction views
- Card grid layout, bid modal overlay, toast notifications

### Admin Page (`admin.html`)
- Dark technical aesthetic — deep navy (`#0f1117`), gold highlights
- Fonts: Syne (UI), Syne Mono (numbers/IDs), Lora (display)
- Sticky topbar with section nav (Items / Live Bids / Winners)
- Table layout with stat cards, confirm modal for destructive actions

### Shared Patterns
- CSS custom properties (`--gold`, `--border`, `--r` etc) for theming
- `.spin` keyframe animation for loaders
- Toast notification system (slide up, auto-dismiss at 3.5s)
- Modal overlay with backdrop blur, ESC-safe close
- `esc()` helper for all user-generated content in innerHTML

---

## Deployment Checklist

- [ ] Paste `Code.gs` into Apps Script bound to a Google Sheet
- [ ] Run `setupSheets()` once to create all tabs
- [ ] Set `AdminPIN` in Script Properties
- [ ] Fill Config sheet (AuctionName, AuctionEnd, WhatsAppNum, SiteURL)
- [ ] Deploy as Web App (Execute as Me, Anyone can access)
- [ ] Paste Web App URL into `const API = '...'` in both HTML files
- [ ] Paste ImgBB API key into `const IMGBB_KEY = '...'` in admin.html
- [ ] Add og:meta tags to index.html head for WhatsApp link previews
- [ ] Host both HTML files on GitHub Pages or Netlify Drop
- [ ] **Redeploy as new version after every Code.gs change**

---

## WhatsApp Integration Points

| Trigger | Message Type |
|---------|-------------|
| Bid modal opens | Contact organiser link (pre-filled item name) |
| Admin: Share button per item row | Single item post (name, description, bid, leader, close date, URL) |
| Admin: Share Top 3 button | Top 3 by bid count (ranked list with bid amounts and leaders) |
| Admin: Winners tab | Per-winner congratulations message with item and amount |

---

## Known Quirks & Gotchas

- Every `Code.gs` change requires a **new deployment version** — saving alone does not update the live URL
- `getConfig` must include `siteUrl` in the returned object for single-item WA share to include the link
- WhatsApp number must be digits only with country code — no spaces, dashes or leading zero (e.g. `27831234567` not `083 123 4567`)
- `AuctionEnd` in Config must be ISO 8601 format: `YYYY-MM-DDTHH:MM:SS`
- ImgBB images are public — do not upload sensitive content
- Apps Script free tier has a 6-minute execution limit and 20k daily URL fetch calls — more than sufficient for community events

