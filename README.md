# TallyCalc

Loan amortization and prepayment modeling in a single self-contained page. No backend, no
analytics, no storage — every figure is computed in the browser and nothing leaves the device.

**Live:** https://grr5000.github.io/tallycalc/

## What it does

- **Fixed-rate loans** — mortgage, auto, personal. Amount, APR, term, first payment date, rolled-in fees.
- **Credit cards** — fixed monthly payment, or a realistic minimum (interest + % of balance with a dollar floor).
- **Prepayments** — recurring extra per month plus any number of one-time lump sums.
- **Recast toggle** — extra money either shortens the term or lowers the payment.
- **Comparison** — interest saved, months saved, and interest saved per $1 of extra paid, always against the no-prepayment baseline.
- **Four charts** — balance, payment split, cumulative interest, yearly breakdown. Canvas-drawn, no chart library.
- **Exports** — CSV of the full schedule, plus a print/PDF view.

Works offline once loaded, and installs to the iOS/Android home screen as a standalone app.

## Repo layout

```
src/                     what actually ships
  index.html             the entire application
  manifest.webmanifest   PWA metadata
  sw.js                  service worker (offline cache)
  icons/                 app icons (180/192/512)
  .nojekyll              tell Pages to skip Jekyll processing
.github/workflows/
  deploy.yml             validate -> build -> deploy
```

Everything is relative-path (`./…`) because a project page is served from `/tallycalc/`,
not from the domain root. Absolute paths like `/sw.js` would 404 — the workflow fails the
build if any sneak in.

## One-time GitHub setup

1. **Settings → Pages → Build and deployment → Source: GitHub Actions.** This is the step
   people miss; leaving it on "Deploy from a branch" makes the workflow run and publish nothing.
2. Push to `main`. The workflow runs automatically.
3. Watch it under the **Actions** tab. First run takes about a minute.

## Pushing this the first time

```bash
cd tallycalc
git init -b main
git remote add origin https://github.com/grr5000/tallycalc.git
git add .
git commit -m "TallyCalc: amortization tool + Pages pipeline"
git push -u origin main
```

## The pipeline

**validate** — fails the build before anything ships if:

- a required file is missing
- the inline JavaScript doesn't parse
- `sw.js` doesn't parse
- `manifest.webmanifest` isn't valid JSON, is missing required keys, or has a non-relative `start_url`
- any absolute-root `href="/…"` or `src="/…"` appears in the HTML

**build** — copies `src/` to `_site/`, adds `.nojekyll`, and rewrites `__BUILD_ID__` in
`sw.js` with the commit SHA and timestamp. That last part matters: it changes the cache name
on every deploy, so returning visitors get the new version instead of a stale cached one.

**deploy** — publishes to Pages via `actions/deploy-pages`.

## Local development

Service workers need a real origin, so `file://` won't register one (the page itself still
works — registration is skipped silently). To test the full PWA:

```bash
cd src && python3 -m http.server 8000
```

Then open http://localhost:8000 — treated as a secure context, so the worker registers.

## Notes on the math

Interest accrues on the period balance at APR ÷ 12. Mortgages generally match this closely;
auto loans and credit cards often accrue daily, so expect small differences from a statement.
Verified against standard cases — $350,000 at 6.25% over 30 years gives a $2,155.01 payment
and $425,803.72 total interest across exactly 360 payments.

Not financial advice. Confirm anything important against your lender's own figures.
