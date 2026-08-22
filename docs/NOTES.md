# TallyCalc — Project Notes

Working notes for the whole build: what it is, how it works, why it's built this way, how it
ships, and every problem hit along the way with its fix. Written to be readable cold, months
from now, with nothing else in front of you.

Last updated: August 2026

---

## 1. Current state

| | |
|---|---|
| **Live site** | https://tallycalc.org |
| **Fallback URL** | https://grr5000.github.io/tallycalc/ (redirects to the custom domain) |
| **Repo** | https://github.com/grr5000/tallycalc |
| **Deploys** | GitHub Actions, automatically on every push to `main` |
| **Registrar** | Namecheap (`tallycalc.org`) |
| **Hosting cost** | $0 — Pages is free on public repos |
| **Domain cost** | ~$8–11/year at renewal |

The entire application is one HTML file. No build step, no framework, no dependencies, no CDN
requests, no backend. Everything computes in the browser; nothing is transmitted or stored.

---

## 2. Quick reference

**Deploy a change**

```bash
# edit src/index.html
git add . && git commit -m "..." && git push      # that's the whole deploy
```

**Test locally with the service worker active** (needed for PWA behavior — `file://` won't register one)

```bash
cd src && python3 -m http.server 8000
# → http://localhost:8000
```

**Validate the way CI does, before pushing**

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('src/index.html','utf8');
[...h.matchAll(/<script(?![^>]*\bsrc=)[^>]*>([\s\S]*?)<\/script>/g)]
.forEach((m,i)=>{try{new Function(m[1])}catch(e){console.error(i,e.message);process.exit(1)}});
console.log('js ok')"
node --check src/sw.js
node -e "JSON.parse(require('fs').readFileSync('src/manifest.webmanifest','utf8'));console.log('manifest ok')"
```

**DNS records at Namecheap** (Domain List → Manage → Advanced DNS)

| Type | Host | Value |
|---|---|---|
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |
| CNAME | `www` | `grr5000.github.io` |

**GitHub settings that must stay set**

- Settings → Pages → Source = **GitHub Actions** (not "Deploy from a branch")
- Settings → Pages → Custom domain = `tallycalc.org`
- Settings → Pages → **Enforce HTTPS** = ticked
- Repo visibility = **Public** (Pages requires a paid plan on private repos)

---

## 3. Repo layout

```
src/                      everything that ships
  index.html              the entire application — markup, CSS, and JS in one file
  manifest.webmanifest    PWA metadata (name, icons, standalone display)
  sw.js                   service worker — offline cache
  CNAME                   custom domain; must survive every deploy
  .nojekyll               stops Pages running Jekyll over the output
  icons/                  180 (iOS home screen), 192, 512 (manifest/maskable)
.github/workflows/
  deploy.yml              validate → build → deploy
docs/
  NOTES.md                this file
README.md                 public-facing description
.gitignore                ignores _site/ build output
```

**Why `src/` rather than files at the repo root:** the deployed site should contain only what
belongs on the web. With a root layout, `README.md` and `docs/` end up published too — and
worse, Jekyll will happily render `README.md` as the homepage if Pages is ever in branch mode.
The `src/` split makes the boundary explicit and lets the workflow assemble exactly what ships.

---

## 4. How `index.html` is organized

Read it in this order — it's roughly top to bottom:

1. **`<head>`** — meta tags (viewport with `viewport-fit=cover`, theme color, apple-mobile-web-app-*),
   manifest link, apple-touch-icon.
2. **`<style>`** — CSS custom properties for the entire palette, then light styling, then a
   `prefers-color-scheme: dark` block that only overrides the variables, then a `max-width: 760px`
   mobile block, then `@media print`.
3. **Markup** — `#mobileTabs`, then `.cols` containing `#paneInputs` and `#paneResults`.
4. **`<script>`** — one inline block, in sections:
   - state and helpers (`$`, `money`, `r2`, `pane`, `isNarrow`)
   - lump-sum row management
   - **the engine** — `pmt()`, `addMonths()`, `amortize()`
   - `run()` — reads inputs, validates, runs the engine twice (baseline + plan), stores in `LAST`
   - `render()` — stat cards, schedule table, then triggers the chart
   - **charts** — `chartColors()`, `tint()`, `drawChart()`, and the pointer/touch tooltip IIFE
   - `exportCSV()`
   - init

**The one global that matters:** `LAST = {base, plan, name, apr, bal, nMonths}`. `base` is the
no-prepayment scenario, `plan` is the one with prepayments applied. Every stat, chart, and export
reads from it. If you're adding a feature, you almost certainly want to read `LAST` rather than
re-run the engine.

---

## 5. The amortization engine

`amortize(opts)` is the whole thing. It walks period by period until the balance clears.

**Options object**

```
balance       starting balance including any rolled-in fees
rate          periodic rate = APR / 100 / 12
nMonths       term in months (fixed mode only)
mode          'fixed' | 'card'
cardRule      'fixed' | 'minpct'
minPct        percent of balance added to interest (card minimum rule)
minFloor      dollar floor for the card minimum
cardPayment   fixed monthly payment (card mode)
extraMonthly  recurring prepayment
extraStart    payment number the recurring extra begins at
lumps         { paymentNumber: amount }
keepPayment   true = extra shortens the term; false = recast, payment recalculated
startDate     ISO date of the first payment
```

**Per period**

1. `interest = balance × rate`, rounded to cents
2. Scheduled payment:
   - fixed mode → the level payment from `pmt()`
   - card, fixed rule → the entered payment
   - card, minimum rule → `max(interest + balance × minPct%, minFloor)`
3. If it's the final scheduled payment of a fixed loan (`k >= nMonths`), the payment is set to
   `balance + interest` so rounding drift can't leave a stray residue
4. Extras applied: recurring (if `k >= extraStart`) plus any lump at `k`
5. Principal trimmed so the balance can't go negative
6. If recast is on (`keepPayment` false) and an extra was applied, the level payment is
   recalculated over the *remaining original term*

**Guards**

- `MAXN = 1200` periods (100 years) — hard stop
- **Negative amortization:** if the scheduled payment doesn't exceed the interest and there's no
  extra to save it, the loop stops and sets `capped: true`. The UI then hides the charts and shows
  a red warning, because a balance that never falls has no meaningful schedule to plot.

**Payment formula** — standard: `P × r / (1 − (1+r)^−n)`, with a `r === 0` branch returning `P / n`
(the formula divides by zero at 0%, which is a real case for promotional financing).

### Verified test cases

Run against a Node harness that extracts and evals the script block. Reproduce any time by
pulling the `<script>` contents out of `index.html` and calling `amortize()` directly.

| Case | Setup | Result |
|---|---|---|
| Standard mortgage | $350,000 · 6.25% · 30yr | Payment **$2,155.01**, exactly **360** periods, interest **$425,803.72**, total paid **$775,803.72** |
| Recurring extra | above + $200/mo | **287** periods, interest **$324,333.90**, saved **$101,469.82** |
| Lump sum | above + $25,000 at payment 13 | **300** periods, interest **$321,276.19**, saved **$104,527.53** |
| Recast | lump with `keepPayment: false` | Still **360** periods, interest **$396,699.60**, payment drops to **$1,999.10** |
| Zero rate | $12,000 · 0% · 24mo | Payment **$500.00**, **$0** interest |
| Card minimum | $8,500 · 24.99% · interest + 1%, $35 floor | **255** periods (21+ years), interest **$16,112.82**, first payment **$262.01** |
| Card fixed | $8,500 · 24.99% · $300/mo | **44** periods, interest **$4,479.50** |
| Negative amortization | $8,500 · 24.99% · $100/mo | Correctly flagged `capped`, warning shown |

Cross-check the first row against any public mortgage calculator — it should match to the cent.

**Known modeling limitation:** interest accrues monthly at APR ÷ 12. Mortgages track this closely.
Auto loans and credit cards frequently accrue daily, so real statements can differ by a few dollars
over the life of the loan. Documented in the UI footnote, not a bug.

---

## 6. Charts

Hand-drawn on a `<canvas>` — no Chart.js, no D3. That was deliberate: a CDN dependency would break
the offline-first goal and add a request to every load, for four fairly simple plots.

Four views, switched by `VIEW`:

| View | What it draws |
|---|---|
| `balance` | Two lines — baseline vs. plan. The gap is the debt prepayments erased. |
| `split` | Stacked area per payment: interest / principal / extra. Shows the crossover point. |
| `cumint` | Cumulative interest, both scenarios; tooltip reports running savings. |
| `yearly` | Stacked bars per loan year. |

**Implementation notes**

- Colors come from CSS variables via `chartColors()`, read at draw time — that's what makes dark
  mode work without a second code path. `tint()` converts a hex var to a translucent `rgba()` fill.
- `drawChart()` bails immediately if the canvas has zero width or height. Required: on mobile the
  results pane is `display: none` until you switch to it, and drawing into a hidden element throws.
- `cv._meta` is stashed after each draw ({n, X, P, pw, ph, bars, series}) so the tooltip handler can
  map a pointer position back to a payment index without recomputing the series.
- Shorter series get padded to full length so a loan paid off early draws a flat line to the end
  rather than stopping mid-chart.
- `dpr` capped at 3 — beyond that you're allocating enormous canvases for no visible gain.

---

## 7. Mobile and PWA

Desktop layout is untouched; everything below is inside `max-width: 760px`.

- **Two-pane tabs.** `#mobileTabs` toggles `body.m-inputs` / `body.m-results` so you're not
  scrolling past a long form to reach results. Calculate auto-switches to results and blurs the
  focused field (otherwise iOS keeps the keyboard up over the answer).
- **16px minimum font on all inputs.** Below that, iOS Safari and Chrome force-zoom on focus. This
  is the single most common reason a form feels broken on iPhone.
- **Touch tooltips.** `touchstart`/`touchmove` mirror the mouse handler; the tooltip pins to the top
  of the chart because a finger covers the point being inspected. `touch-action: pan-y` on the
  canvas keeps vertical page scrolling working while horizontal drags track the chart.
- **Table columns 3 and 8** (Payment, Cumulative interest) are hidden on narrow screens — both are
  derivable from the remaining columns — and restored in print.
- **Safe-area insets** on header and wrap for the notch and home indicator.
- **Service worker** registers only on `https:` or `localhost`, so opening the file directly still
  works with registration silently skipped.

**Cache strategy** (`sw.js`): navigations are network-first falling back to cache — a new deploy
appears immediately when online, and the app still opens offline. Everything else is cache-first.
`CACHE` contains `__BUILD_ID__`, which the workflow rewrites with the commit SHA, so each deploy
invalidates the previous cache automatically. Without that, users get stuck on a stale version
indefinitely — the classic service worker trap.

---

## 8. The deploy pipeline

`.github/workflows/deploy.yml`, three jobs, on push to `main`.

**validate** — nothing ships if any of these fail:

- a required file is missing (`index.html`, manifest, `sw.js`, `CNAME`, all three icons)
- `CNAME` doesn't contain exactly `tallycalc.org` on exactly one line
- the inline JavaScript doesn't parse (extracted and run through `new Function`)
- `sw.js` doesn't parse (`node --check`)
- the manifest isn't valid JSON, is missing a required key, or has a non-relative `start_url`
- any absolute-root `href="/…"` or `src="/…"` appears in the HTML

**build** — copies `src/` → `_site/`, adds `.nojekyll`, rewrites `__BUILD_ID__` with
`${GITHUB_SHA::7}` plus a UTC timestamp.

**deploy** — `actions/deploy-pages@v4`.

**Why the absolute-path check exists:** the site originally lived at `grr5000.github.io/tallycalc/`,
a subpath. An absolute `/sw.js` resolves to the domain root and 404s there while working fine in
local testing. Now that a custom domain serves from the root, the check is less critical — but
keeping paths relative means the site can move anywhere without edits, which is worth preserving.

**Why `enablement: true` on `configure-pages`:** lets the workflow turn Pages on via the API rather
than assuming it exists. Note it does *not* override the source setting if someone has switched it
to branch mode in Settings — see §9.

---

## 9. Troubleshooting log

Everything that went wrong, in order, with the actual cause.

**HTML file emailed to a phone opens as a static preview, buttons dead**
Mail and Files render HTML through Quick Look, which deliberately doesn't execute JavaScript. iOS
Safari also cannot open `file://` URLs at all. There is no workaround — the whole tool is JS. This
is what drove the decision to host it. (Android Chrome *can* open local HTML files fine.)

**`git push` rejected — "Updates were rejected because the remote contains work"**
The repo was created on GitHub with a README, so remote had a commit the local `git init` history
never contained. Two unrelated histories. Resolved by force-pushing over the throwaway README:
`git fetch origin && git push --force-with-lease origin main`.

**`--force-with-lease` rejected — "stale info"**
The lease needs a current remote-tracking ref to compare against, and `origin/main` had never been
fetched. `git fetch origin` first, then the push works. This is the flag doing its job, not a bug.

**Actions run fails — `Get Pages site failed... Not Found`**
Pages wasn't enabled on the repo yet, so the Pages API returned 404. Fix: Settings → Pages →
Source → GitHub Actions, then re-run the workflow.

**Warning — "Node.js 20 is deprecated"**
`actions/checkout@v4` shipped a Node 20 runtime and runners now force it to Node 24. Harmless, but
bumped to `actions/checkout@v5`. `configure-pages@v5` is already the latest major; GitHub is
rolling Node 24 into that same tag.

**Custom domain serves the README instead of the app**
The giveaway was `<meta name="generator" content="Jekyll v3.10.0">` in the served HTML. Pages
source had reverted to "Deploy from a branch", so Jekyll built the repo and turned `README.md` into
the homepage. Fix: Settings → Pages → Source → GitHub Actions, then re-run. **If the site ever
shows the README again, check this setting first** — enabling Pages through the Settings UI
defaults to branch mode and silently overrides whatever was there.

**CI step failed — `require('./src/manifest.webmanifest')` threw `Unexpected token ':'`**
Node's `require` dispatches on file extension and doesn't know `.webmanifest`, so it tried to parse
JSON as JavaScript. Switched to `JSON.parse(fs.readFileSync(...))`.

### Engine bugs found during verification

**A 361st payment appeared on a 360-month loan.** Cent-rounding drift across 360 periods left a few
dollars outstanding. Fixed by forcing the final scheduled payment of a fixed loan to
`balance + interest`.

**Credit card minimums never paid off.** The original rule was a flat percent of the balance; at
24.99% APR, 2% of the balance is *less than* the monthly interest, so the balance rose forever.
Rewritten to the actual US issuer convention — interest plus a percent of principal, with a dollar
floor — which pays off in a realistic 255 months.

---

## 10. Analytics

GitHub Pages provides **no** server logs and no visitor statistics — before this was added, every
visit was simply unrecorded, with no way to recover the history. (Repo Insights → Traffic counts
views of the github.com repo page, not the site.)

**What's installed:** Cloudflare Web Analytics, added as a small block near the end of the inline
script in `index.html`. It is cookieless, needs no consent banner, and records page views only —
it never sees anything entered into the calculator.

**Setup:** dash.cloudflare.com → Web Analytics → Add a site → `tallycalc.org` → copy the token →
paste it over `PASTE_TOKEN_HERE` in `index.html`. The block is a no-op until a real token is
present, and it skips non-HTTPS origins so local testing never pollutes the numbers. CI emits a
warning (not a failure) while the placeholder is still there.

**Dashboard:** dash.cloudflare.com → Web Analytics. Free tier caps most reports at the top 15
entries per category.

**Expect undercounting.** Ad blockers and Safari's tracking prevention block the beacon, and an
installed PWA opened offline never phones home. Treat the number as a floor, not a census.

**Privacy wording:** the header deliberately says *"your loan figures... never leave your device"*
rather than the original *"nothing is sent anywhere."* The narrower claim is the one that stays
true with analytics present. If analytics is ever removed, the broader wording can come back.
There's a matching Privacy section in `README.md`.

---

## 11. Ideas if picking this back up

- **Save/load loans** — deliberately omitted (chose single-session). `localStorage` would be the
  obvious add; scope it to the origin and note it makes the page no longer stateless.
- **Extra payment frequency** — biweekly and semi-monthly are common and would need the period
  loop generalized beyond monthly.
- **APR vs. daily accrual toggle** — would close the documented gap with card and auto statements.
- **Comparison of two scenarios side by side** — the engine already runs twice; a third run is cheap.
- **Amortization for adjustable rates** — a rate schedule keyed by payment number would slot into
  the existing loop without restructuring.

---

## 12. If something breaks

1. **Site shows the README** → Pages source flipped to branch mode. §9.
2. **Custom domain stopped working** → check `src/CNAME` still exists and Settings → Pages still
   lists the domain. CI fails the build if the file is missing, so a green build means the file shipped.
3. **Changes don't appear on the phone** → the service worker served a cached copy. Confirm the
   Actions run succeeded, then hard-reload; the build-ID cache busting should make this rare.
4. **Charts blank on mobile** → almost certainly a zero-size canvas draw. Check the pane-switch
   redraw timing in `pane()`.
5. **Numbers look wrong** → run the §5 test cases first. If those pass, the engine is fine and the
   bug is in input parsing or rendering.
