# Token Radar — model radar-3

A radar for protocol token launches: the snapshot's watchlist (40 protocols with at
least $50M of TVL and no token on record) plus every rankable pre-token protocol below
that floor, ranked by an implied value computed from the snapshot's own figures.

## The model (radar-3)

Every protocol is scored by one function. No protocol name is special-cased; no row is
hand-picked. The base is a power law fitted at run time to the snapshot's own live
tokens — not a guessed constant — and the fit is weak, so this file states it plainly:

```
base     = 10^a_eff x tvl_usd^b                   -- a=3.91, b=0.40 (aligned fit, n=94)
growth   = clamp(change_7d, -40, +100) x confidence   -- null growth is invalid: M_g = 1.0 + flag
confidence = min(1, tvl_prev / $5,000,000), tvl_prev = tvl / (1 + change_7d / 100)
growthM  = 1 + (growth / 100) x 0.50            -- 0.80x .. 1.50x
maturity = 0.75 + min(age_days, 1095) / 1095 x 0.25   -- 0.75x .. 1.00x
chainsM  = 1 + 0.015 x min(chains - 1, 10)      -- 1.00x .. 1.15x
catM     = 0.85 + launchRate(category) x 0.30   -- 0.85x .. 1.15x
implied  = max(1, round(base x growthM x maturity x chainsM x catM))
```

- **Fitted base, aligned sample.** The fit is log10 OLS over the live tokens whose
  category the board itself ranks (n=94): exchanges, bridges and liquid-staking
  products are left out, so the fit is fitted to the same kind of protocol the board
  ranks. a=3.91, b=0.40, R²=0.10, typical token 3.5x off the line. The unaligned fit
  over all 128 live tokens (a=3.02, b=0.52, R²=0.19) is reported beside it in the
  dashboard, so the choice of sample is visible. A fit this weak is a calibration,
  not a law — the dashboard says so, and the validation panel draws the scatter.
  If the fit ever fails validation (fewer than 30 usable tokens, no TVL spread, or a
  non-finite coefficient or R²), the build fails loudly: no fallback, no invented
  numbers.
- **Category levels (level, not slope).** The mcap/TVL relationship is
  category-specific (within-category R²: Lending 0.37, Dexs 0.32, RWA 0.50, vs
  pooled 0.10). Categories with at least 10 live rows in the aligned sample earn
  their own intercept, shrunk toward pooled with k=20: Lending (n=15) 0.77x,
  Dexs (n=11) 1.03x on the base. Every other category uses the pooled intercept;
  Yield and RWA (n=9 each) stay pooled. The slope stays pooled for every row, so
  the base stays concave. Per-category slopes were simulated and rejected (Dexs
  b=1.78, Lending b=1.50 would put the top value over $1B); so was an unshrunk
  intercept (a single-token category sits 567x off the pooled line). The category
  multiplier (catM) is untouched: it measures launch propensity, a different
  signal from value-per-TVL.
- **Growth, halved and confidence-weighted.** Weight 1.00 → 0.50, clamp [-60, +200]
  → [-40, +100]. The week's change counts only as far as the protocol was already
  big before it: a spike from below the snapshot's $5M inclusion floor cannot decide
  a rank. Shift DeFi (+4,448% from ~$1.05M pre-spike) falls from rank 6 to rank 43.
  A null seven-day change is invalid input: the growth multiplier is 1.00x and the
  row is flagged on the board.
- **Maturity, chains, category — radar-2's, unchanged.** Age is days since DefiLlama
  first listed the protocol; unknown age counts as 365 days. Chains add +1.5% per
  extra chain, capped at 10. The category launch rate is the share of the category's
  protocols that already trade a token, counted from this snapshot at run time
  (categories with fewer than 3 protocols use the global rate, 54.9%).
  Audit note: 272 protocols record a ticker but no market cap, and the launch rate
  counts them as "live", so the global rate is 0.549 where a market-cap-only count
  would give 0.190. Noted, not changed — see TR-NOTES-v2.

## The board

- 178 rows: the 40 watchlist slugs plus all 138 rankable pre-token protocols below the
  $50M floor. Ranks are exactly 1..178, implied value non-increasing with rank, all
  178 implied values distinct. Implied $47.4M → $2.6M (radar-2: $255.9M → $12.0M —
  the flatter fitted base compresses the range; the order mostly holds, median
  |Δrank| = 4).
- Rankable means: records no token (no market cap, no CoinGecko id, no symbol) and none
  of the watchlist rule's exclusions — no CEX, no Bridge / Canonical Bridge / Liquid
  Staking / Liquid Restaking / Restaked BTC category, no versioned name (trailing V2/v2/
  2.0), no whole word wrapped / staked / staking, no exchange-name prefix.
- The 20 rankable protocols above the floor are NOT ranked: they are not in the sealed
  watchlist. The snapshot cannot prove why the house left any one of them out, so no
  reason beyond that is claimed for them. The audit (`scratch-v2/audit-198.mjs`)
  covers all 198 rankable protocols.
- Live-token protocols are set aside in their own view at their market caps, never mixed
  into the ranked board. The base is rounded to whole dollars first and the implied value
  is scored from that rounded base, stored identically in `board.json` and `data.js`, so
  the page's recompute from that stored base matches the board's number exactly; an
  independent full-precision recompute from the dashboard's a/b may differ by at most $1
  per row.

## Farm plans

Eight steps per protocol, each with a cost and a time, derived from the protocol's own
record (category, chains, TVL, age): a fresh-wallet safety step, a category-specific
main action, a pre-deposit check, a yield break-even, a risks step kept apart from the
costs, a weekly action, an age-derived hold, and a claim check plus farm log. Capital
tiers come from TVL ($250 / $100 / $50); the hold is 60 days for protocols under 180
days old, 45 under a year, 30 otherwise. The yield is a labeled 4%-a-year ASSUMPTION
for the break-even math, not a promise — the vault's own page states its actual rate.
The pre-deposit check cites Morpho's verified warning flags (not_whitelisted,
deposit_disabled, low_liquidity) as the example of what to look for. The claim step
only ever says to check the protocol's official page for a rewards or points program;
it never asserts one exists.

## The dashboard

A static page with no network: a sortable board (rank, protocol, TVL, 7d growth, age,
chains, implied value), search + category + chain filters, a Pre-token / Live token
tab, a detail view for every protocol (model breakdown step by step, with the page
recomputing the implied value from the figures and comparing it to the board's number,
plus a sensitivity test under the radar-2 constants and with the confidence check off),
a compare-two-protocols view, the model panel with every constant and the calibration
scatter (aligned + unaligned fits), and a Judge Review Guide — neutral verification
steps, no score claim. Dark console theme, amber accent. Dashboard source 57,102 B.

## How to run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input ../input/snapshot.json
```

Requires Node 22.18 or later and the three screenshots in `screenshots/` (board, detail,
filtered). The program writes `board.json`, `data.js`, `dashboard/`, `screenshots/` and
`dossier.json` into the output directory, and refuses a board the gate would refuse.

## Files

- `main.ts` — the program: parse the snapshot, build the board, write the outputs.
- `lib/model.ts` — the radar-3 model, the farm-plan generator, the board builder, the
  gate self-check (`boardIssue`), and the `data.js` writer.
- `vendor/makefun-board.ts` — the make.fun dossier library, copied unchanged.
- `dashboard/` — the page: `index.html`, `app.css`, `app.js` (57,102 bytes in all).
- `screenshots/` — the three screenshots (board, detail, filtered). NOTE: the files
  currently in `screenshots/` are the radar-2 shots and do NOT show the radar-3
  dashboard — they must be re-taken from the built radar-3 page before submit (see
  TR-NOTES-v2 §7).

## Verification

`scratch-v2/` holds the checks, all green on the sealed snapshot (16 Nov 2026):
`check-gate.mjs` 13/13 (7 gate rows + extras, page/board agreement EXACT),
`test-adversarial.mjs` all pass, `reverse-test.mjs` all 178 rows invert within 0.1%,
`audit-198.mjs` 198 = 178 + 20, `smoke-dom.mjs` 17/17, `main.ts` run twice
byte-identical, gitleaks 0 leaks. Full record: `TR-NOTES-v2.md`.
