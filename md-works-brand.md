# MD Works — Brand & Design System
## Morney Deetlefs · Personal Brand Reference

> Paste this file at the start of any new chat to instantly apply the MD Works brand to any project.

---

## Identity

| | |
|---|---|
| **Brand Name** | MD Works |
| **Full Name** | Morney Deetlefs |
| **Location** | South Africa |
| **Tagline** | Builder of useful things for real people |
| **Domains** | silentauction.morneydeetlefs.workers.dev (current) |
| **Mark** | ✦ (used as ornament, divider, brand symbol throughout) |

### Discipline Tags
Community Tech · Full-Stack · Creative Tools · Social Good

### Core Values (shown in portfolio)
1. Zero-cost first — target R0 running cost using free-tier services
2. Real deployment — everything ships live, used by real people
3. Maintainable by anyone — Google Sheets means non-devs can update content
4. Craft over speed — every detail considered, no shortcuts on UX

---

## Design Tokens

Copy this block into any project's `<style>` tag or `brand.css` file:

```css
/* ══════════════════════════════════════════
   MD WORKS — BRAND TOKENS v1.0
   Morney Deetlefs · mdworks.dev
══════════════════════════════════════════ */
:root {
  /* Backgrounds */
  --md-black:      #0a0906;   /* deepest background */
  --md-dark:       #110e09;   /* primary page background */
  --md-surface:    #1a1610;   /* cards, panels */
  --md-panel:      #211c14;   /* elevated panels */
  --md-border:     #2c2619;   /* subtle borders */
  --md-border-lt:  #3d3526;   /* lighter borders, hover states */
  --md-faint:      #3d3526;   /* very subtle text */

  /* Gold scale — the signature */
  --md-gold-dk:    #7a5815;   /* muted, aged gold — decorative */
  --md-gold:       #c9943c;   /* primary gold accent */
  --md-gold-lt:    #e8c87a;   /* light gold — headings, highlights */
  --md-gold-pale:  #f5e4bb;   /* palest gold — rarely used */
  --md-gold-glow:  rgba(201,148,60,.12); /* ambient glow */

  /* Text */
  --md-cream:      #f0e6ce;   /* primary text */
  --md-muted:      #7a6d58;   /* secondary / helper text */

  /* Semantic */
  --md-success:    #4caf7a;
  --md-danger:     #c94c4c;
  --md-info:       #4c8fc9;

  /* Geometry */
  --md-radius:     4px;
  --md-radius-sm:  2px;
  --md-radius-lg:  8px;
}
```

---

## Typography System

### Font Stack
```html
<!-- Always include this in <head> -->
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@400;600;900&family=Cormorant+Garamond:ital,wght@0,300;0,400;0,600;1,300;1,400&family=Raleway:wght@200;300;400;500&family=Syne:wght@400;500;600;700&family=Syne+Mono&display=swap" rel="stylesheet"/>
```

### Roles

| Role | Font | Weight | Use |
|------|------|--------|-----|
| **Display / Logo** | Cinzel | 900 | Page titles, brand mark, hero headings |
| **Headings** | Cinzel | 400–600 | Section titles, card names, nav brand |
| **Editorial** | Cormorant Garamond | 300–400 italic | Taglines, callouts, marketing copy, pull quotes |
| **UI Body** | Raleway | 300–500 | Body text, labels, buttons (public-facing pages) |
| **Admin UI** | Syne | 400–700 | Navigation, admin interfaces, dashboard text |
| **Monospace** | Syne Mono | 400 | IDs, numbers, code, timestamps, stat values |

### Typography Rules
- Cinzel is the strongest brand signal — use it on every page at least once
- Cormorant Garamond italic for any line that needs warmth or humanity
- Never use Inter, Roboto, Arial or system fonts — breaks brand immediately
- Heading colour: `var(--md-gold-lt)` on dark backgrounds
- Body text: `var(--md-cream)` at full weight, `rgba(240,230,206,.7)` at reduced weight
- Labels / eyebrows: Syne Mono, `.65rem`, `letter-spacing: .3em`, `text-transform: uppercase`, `color: var(--md-gold-dk)`

---

## Colour Usage Rules

### Do
- `--md-gold` as the primary accent on interactive elements, borders on hover
- `--md-gold-lt` for headings and important values
- `--md-gold-dk` for decorative elements, ornaments, disabled states
- `--md-surface` for cards — never pure black
- Gold gradient: `linear-gradient(135deg, var(--md-gold-dk), var(--md-gold))` for primary buttons and highlighted elements

### Don't
- Never use pure white — always `--md-cream` or `--md-gold-pale`
- Never use blue as a primary accent — it breaks the warm palette
- Never use purple — cliché and off-brand
- Avoid bright colours — everything is muted and warm

### Gold Gradient (primary CTA)
```css
background: linear-gradient(135deg, var(--md-gold-dk), var(--md-gold));
color: var(--md-black);
```

### Gold Text Gradient (hero titles)
```css
background: linear-gradient(135deg, var(--md-gold-lt) 0%, var(--md-gold) 50%, var(--md-gold-dk) 100%);
-webkit-background-clip: text;
-webkit-text-fill-color: transparent;
background-clip: text;
```

### Outlined Text (secondary hero line)
```css
color: transparent;
-webkit-text-stroke: 1px var(--md-gold-dk);
```

---

## Component Patterns

### Grain Texture Overlay (every page)
```css
body::before {
  content: '';
  position: fixed; inset: 0; z-index: 0;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='200' height='200'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='200' height='200' filter='url(%23n)' opacity='0.035'/%3E%3C/svg%3E");
  pointer-events: none;
}
```

### Ambient Glow (hero pages)
```css
body::after {
  content: '';
  position: fixed;
  top: -20vh; left: 50%;
  transform: translateX(-50%);
  width: 80vw; height: 60vh;
  background: radial-gradient(ellipse, rgba(201,148,60,.06) 0%, transparent 70%);
  pointer-events: none; z-index: 0;
}
```

### Section Eyebrow Label
```css
.section-label {
  font-family: 'Syne Mono', monospace;
  font-size: .62rem; letter-spacing: .3em; text-transform: uppercase;
  color: var(--md-gold-dk);
  display: flex; align-items: center; gap: 1rem;
}
.section-label::after {
  content: ''; flex: 1; max-width: 60px; height: 1px;
  background: var(--md-border-lt);
}
```

### Full-Width Divider with Ornament
```css
.full-divider {
  width: 100%; height: 1px;
  background: linear-gradient(90deg, transparent 0%, var(--md-border-lt) 20%,
    var(--md-gold-dk) 50%, var(--md-border-lt) 80%, transparent 100%);
  display: flex; align-items: center; justify-content: center;
  position: relative;
}
.full-divider::after {
  content: '✦';
  position: absolute;
  font-size: .7rem; color: var(--md-gold-dk);
  background: var(--md-black); /* match page bg */
  padding: 0 1rem;
}
```

### Card
```css
.card {
  background: var(--md-surface);
  border: 1px solid var(--md-border);
  border-radius: var(--md-radius);
  transition: border-color .25s, transform .2s;
}
.card:hover {
  border-color: var(--md-gold-dk);
  transform: translateY(-3px);
}
```

### Card Gold Glow on Hover
```css
.card { position: relative; }
.card::before {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(135deg, var(--md-gold-glow) 0%, transparent 60%);
  opacity: 0; transition: opacity .3s;
  pointer-events: none;
  border-radius: inherit;
}
.card:hover::before { opacity: 1; }
```

### Primary Button
```css
.btn-primary {
  padding: .8rem 1.6rem;
  background: linear-gradient(135deg, var(--md-gold-dk), var(--md-gold));
  color: var(--md-black);
  border: none; border-radius: var(--md-radius);
  font-family: 'Cinzel', serif;
  font-size: .8rem; font-weight: 600; letter-spacing: .12em;
  cursor: pointer; transition: opacity .2s, transform .1s;
}
.btn-primary:hover { opacity: .9; }
.btn-primary:active { transform: scale(.98); }
```

### Secondary Button
```css
.btn-secondary {
  padding: .75rem 1.4rem;
  background: transparent;
  border: 1px solid var(--md-border-lt);
  color: var(--md-muted);
  border-radius: var(--md-radius);
  font-family: 'Raleway', sans-serif;
  font-size: .82rem; letter-spacing: .08em;
  cursor: pointer; transition: all .2s;
}
.btn-secondary:hover { color: var(--md-gold-lt); border-color: var(--md-gold-dk); }
```

### Toast Notification
```css
.toast {
  position: fixed; bottom: 1.5rem; left: 50%; z-index: 200;
  transform: translateX(-50%) translateY(80px);
  background: var(--md-surface);
  border-radius: 50px;
  padding: .7rem 1.4rem;
  font-size: .85rem; letter-spacing: .05em;
  box-shadow: 0 8px 32px rgba(0,0,0,.5);
  transition: transform .35s cubic-bezier(.34,1.56,.64,1);
  white-space: nowrap; border: 1px solid var(--md-border);
}
.toast.show { transform: translateX(-50%) translateY(0); }
.toast.success { border-color: var(--md-success); color: var(--md-success); }
.toast.error   { border-color: var(--md-danger);  color: var(--md-danger); }
```

### Loader Spinner
```css
.loader {
  display: flex; align-items: center; justify-content: center;
  padding: 4rem; gap: .8rem;
  color: var(--md-muted); font-size: .85rem;
}
.spin {
  width: 20px; height: 20px;
  border: 2px solid var(--md-border);
  border-top-color: var(--md-gold);
  border-radius: 50%;
  animation: spin .8s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
```

### Scroll Reveal (JS + CSS)
```css
.reveal {
  opacity: 0; transform: translateY(30px);
  transition: opacity .7s ease, transform .7s ease;
}
.reveal.visible { opacity: 1; transform: translateY(0); }
```
```javascript
const revealObserver = new IntersectionObserver(entries => {
  entries.forEach((entry, i) => {
    if (entry.isIntersecting) {
      setTimeout(() => entry.target.classList.add('visible'), i * 60);
      revealObserver.unobserve(entry.target);
    }
  });
}, { threshold: 0.1 });
document.querySelectorAll('.reveal').forEach(el => revealObserver.observe(el));
```

### Custom Gold Cursor (portfolio / showcase pages)
```css
.cursor {
  position: fixed; z-index: 9999;
  width: 8px; height: 8px;
  background: var(--md-gold);
  border-radius: 50%; pointer-events: none;
  transform: translate(-50%, -50%);
  transition: width .2s, height .2s, opacity .2s;
  mix-blend-mode: screen;
}
.cursor.large {
  width: 40px; height: 40px;
  background: transparent;
  border: 1px solid var(--md-gold); opacity: .5;
}
```
```javascript
const cursor = document.getElementById('cursor');
let mx=0, my=0, cx=0, cy=0;
document.addEventListener('mousemove', e => { mx=e.clientX; my=e.clientY; });
(function animate() {
  cx += (mx-cx)*.15; cy += (my-cy)*.15;
  cursor.style.left = cx+'px'; cursor.style.top = cy+'px';
  requestAnimationFrame(animate);
})();
document.querySelectorAll('a,button,.card').forEach(el => {
  el.addEventListener('mouseenter', ()=>cursor.classList.add('large'));
  el.addEventListener('mouseleave', ()=>cursor.classList.remove('large'));
});
```

---

## WhatsApp Message Rules

All WA messages across MD Works projects follow these rules — no exceptions:

- **No emoji** — renders as black diamonds on Windows Chrome via wa.me links
- Use WhatsApp markdown only:
  - `*text*` for bold
  - `_text_` for italic
  - `----` for horizontal dividers
- These are processed by WhatsApp after paste — work on all platforms
- Numbers/currency always in ZAR: `R 250.00`
- Always end share messages with a site URL on its own line

---

## Footer Standard

Every MD Works project gets this footer:

```html
<footer style="text-align:center;padding:2.5rem;border-top:1px solid var(--md-border)">
  <div style="font-family:'Cinzel',serif;font-size:.75rem;letter-spacing:.22em;
    text-transform:uppercase;color:var(--md-gold-dk);margin-bottom:.5rem">
    ✦ &nbsp; MD Works &nbsp; ✦
  </div>
  <div style="font-family:'Syne Mono',monospace;font-size:.6rem;
    letter-spacing:.14em;color:var(--md-faint)">
    Morney Deetlefs · South Africa · Builder of useful things for real people
  </div>
</footer>
```

---

## Page Templates (quick-start)

### Dark Page Shell
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>Page Title — MD Works</title>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@400;600;900&family=Cormorant+Garamond:ital,wght@0,300;0,400;1,400&family=Raleway:wght@300;400;500&family=Syne+Mono&display=swap" rel="stylesheet"/>
<style>
  /* PASTE MD WORKS TOKENS HERE */
  *, *::before, *::after { box-sizing:border-box; margin:0; padding:0; }
  body { font-family:'Raleway',sans-serif; background:var(--md-dark); color:var(--md-cream); min-height:100vh; overflow-x:hidden; }
  body::before { /* grain texture */ }
</style>
</head>
<body>
  <!-- content -->
  <!-- PASTE MD WORKS FOOTER HERE -->
</body>
</html>
```

### Admin Shell (technical aesthetic)
Same as above but swap Raleway for Syne as body font, and use `--md-black` as background.

---

## Existing Projects

| Project | URL | Files |
|---------|-----|-------|
| Silent Auction + Tombola | https://silentauction.morneydeetlefs.workers.dev/ | index.html |
| Admin Dashboard | https://silentauction.morneydeetlefs.workers.dev/admin.html | admin.html |
| Admin Help Guide | https://silentauction.morneydeetlefs.workers.dev/admin-help.html | admin-help.html |
| Sponsor Proposal Generator | https://silentauction.morneydeetlefs.workers.dev/proposal-generator.html | proposal-generator.html |
| Portfolio | https://silentauction.morneydeetlefs.workers.dev/portfolio.html | portfolio.html |

---

## Session Opener for New Chats

Paste this at the start of any new chat along with this file:

> *"I'm Morney Deetlefs (MD Works), based in South Africa. I build zero-cost community web tools using vanilla HTML/JS, Google Apps Script and Google Sheets. My brand uses a dark gold aesthetic — full token system and component library is in the attached md-works-brand.md. Please apply this brand to everything we build."*

---

## Quick Reference — What to Always Include

| Element | Value |
|---------|-------|
| Page background | `#110e09` or `#0a0906` |
| Card background | `#1a1610` |
| Primary accent | `#c9943c` |
| Heading colour | `#e8c87a` |
| Body text | `#f0e6ce` |
| Muted text | `#7a6d58` |
| Display font | Cinzel |
| Body font | Raleway (public) / Syne (admin) |
| Mono font | Syne Mono |
| Brand ornament | ✦ |
| Currency | R (ZAR) |
| WA messages | No emoji, WhatsApp markdown only |
| Footer | ✦ MD Works ✦ · Morney Deetlefs · South Africa |

