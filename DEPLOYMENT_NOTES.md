# Deployment Notes — Packed Like Sardines Case Study
## Hard lessons from the March 2026 session. Read this before every push.

---

## VERCEL BEHAVIOUR — what we learned the hard way

### The dashboard thumbnail is always stale
Vercel generates a static screenshot of each deployment once and never refreshes it.
The thumbnail you see in the dashboard is NOT the live site. Always verify by visiting
the actual URL. Never trust the thumbnail.

### Auto-deploy IS working — every push to main triggers production
The GitHub integration fires on every push. Each commit creates a new production
deployment at sardinecasestudy.vercel.app. The confusion came from the stale thumbnail.

### How to verify what's actually live
```bash
curl -s "https://sardinecasestudy.vercel.app" | grep "YOUR_EXPECTED_TEXT"
```
This is the only reliable way to confirm content. Use it every time.

### Cache busting for browser
Add a query param to force a fresh browser fetch: `?v=X` with any value.
The server ignores it; the browser treats it as a new URL and bypasses cache.
Hard refresh (Cmd+Shift+R) works too but is less reliable across CDN edges.

---

## BAR CHART — the bugs that kept breaking it

### NEVER use JS to set bar widths. Set them statically in HTML.
Every attempt to use JavaScript (IntersectionObserver, setTimeout fallback, data-w
attributes) eventually broke the chart. The only reliable approach:

```html
<!-- CORRECT — static inline width, no JS needed -->
<div class="bar-fill artisan" style="width:87%"><span class="bar-val">$12.99</span></div>

<!-- WRONG — data-w + JS fallback -->
<div class="bar-fill artisan" style="width:0%" data-w="87%">...</div>
```

### NEVER use `position:absolute` on bar fills
`.bar-fill { position: absolute; }` with `width:%` can behave unexpectedly inside
flex/grid containers. Use normal flow instead:

```css
/* CORRECT */
.bar-fill { height: 100%; border-radius: 2px; display: flex; ... }
.bar-track { flex: 1; height: 32px; border-radius: 2px; overflow: hidden; }

/* WRONG */
.bar-fill { position: absolute; left: 0; top: 0; ... }
.bar-track { position: relative; ... }
```

### The `|| '50%'` fallback will destroy your chart
If you ever write: `bar.style.width = bar.getAttribute('data-w') || '50%'`
and the data-w attributes don't exist, every bar becomes exactly 50%.
This is what happened. The fix: delete all bar animation JS entirely.

### Partial regex replacements leave dangling `}` characters
When we removed the `revealBars` function text via regex but left the closing `}`,
it created a JS syntax error. The setTimeout fallback still ran and executed the
`|| '50%'` override. Always remove the entire script block cleanly.

---

## CONTENT ACCURACY — always verify the HTML matches what you built elsewhere

### The positioning matrix had wrong brands for months
The deployed matrix showed Spanish Galician canners (Los Peperetes, Ramón Peña,
Pujado Solano, La Curiosa) — brands not sold in Canada — instead of the correct
NA retail competitors from the design deck.

**Correct brands for the NA market positioning matrix:**
- Premium × Traditional: La Belle Iloise, Nuri, Patagonia, La Brujula
- Premium × Modern (target quadrant): BELA, Fishwife, Scout Canning, Nice Cans
- Value × Modern: Riga Gold
- PLS target: gold dot in Premium × Modern quadrant

### Always cross-check deployed content against source documents
When integrating a PPT/PDF deck, grep the HTML for key brand names after pushing
to confirm the right content made it in.

---

## GIT WORKFLOW — keep it clean

### Check git status before pushing
```bash
cd /tmp/sardinecasestudy
git status           # confirm what's staged
git diff --stat HEAD # confirm what changed
git log --oneline -5 # confirm commit message
```

### The repo resets between Claude sessions — always pull first
```bash
git fetch origin && git checkout origin/main -- packed_like_sardines_case_study.html
```
Then apply patches on top of the latest GitHub version, not a stale local copy.

### GitHub token in use
ghp_[REVOKED — see your GitHub token settings] — REVOKE this at github.com →
Settings → Developer settings → Personal access tokens. It appeared in conversation
history and should be treated as compromised.

---

## QUICK VERIFICATION CHECKLIST before any push

```bash
# 1. Confirm bar widths are static inline (not 0%)
grep -c 'style="width:0%"' packed_like_sardines_case_study.html
# Should return 0

# 2. Confirm no data-w attributes remain
grep -c 'data-w=' packed_like_sardines_case_study.html
# Should return 0

# 3. Confirm no bar animation JS remains
grep -c 'revealBars\|barObserver\|data-w\||| .50%' packed_like_sardines_case_study.html
# Should return 0

# 4. Confirm key content is present
grep -c '01.5\|Design intent\|Fishwife\|Carmelo\|Güeyu' packed_like_sardines_case_study.html
# Should return 5

# 5. After push, verify live
curl -s "https://sardinecasestudy.vercel.app" | grep "01.5\|Fishwife\|Nice Cans"
```
