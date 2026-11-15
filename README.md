# Token Radar — model radar-2

A radar for protocol token launches: the snapshot's watchlist (40 protocols with at
least $50M of TVL and no token on record) plus every rankable pre-token protocol below
that floor, ranked by an implied value computed from the snapshot's own figures.

## The model (radar-2)

Every protocol is scored by one function. No protocol name is special-cased; no row is
hand-picked. The constants below are the whole model:

```
base     = $120,000,000 x log10(1 + tvl_usd / $10,000,000)
growth   = clamp(change_7d, -60, +200)          -- null growth counts as 0
growthM  = 1 + (growth / 100) x 1.00            -- 0.40x .. 3.00x
maturity = 0.75 + min(age_days, 1095) / 1095 x 0.25   -- 0.75x .. 1.00x
chainsM  = 1 + 0.015 x min(chains - 1, 10)      -- 1.00x .. 1.15x
catM     = 0.85 + launchRate(category) x 0.30   -- 0.85x .. 1.15x
implied  = max(1, round(base x growthM x maturity x chainsM x catM))
```

- **Log base.** TVL is dampened with a logarithm: a $2.75B protocol scores about 3x a
  $50M one, not 55x. A token launch is not worth 55x more just because the vault is 55x
  bigger.
- **Growth.** Seven-day TVL change, clamped to [-60%, +200%], with weight 1.00 — strong
  enough to actually move ranks. A protocol growing +150% in a week can outrank a flat
  protocol ten times its size. The clamp stops one wild week (the snapshot's largest
  change is +4,448%) from deciding a rank.
- **Maturity.** Age is days since DefiLlama first listed the protocol, the best record of
  how long it has run with no token. Unknown age counts as 365 days.
- **Chains.** More chains, slightly more value: +1.5% per extra chain, capped at 10.
- **Category.** The launch rate is the share of the category's protocols that already
  trade a token, counted from this snapshot at run time (categories with fewer than 3
  protocols use the global rate, 54.9%). A pre-token DEX sits in a category where 74% of
  protocols already trade; a pre-token Risk Curator sits where only 13% do. The factor
  is a snapshot statistic, not a hand-written list.

## The board

- 178 rows: the 40 watchlist slugs plus all 138 rankable pre-token protocols below the
  $50M floor. Ranks are exactly 1..178, implied value non-increasing with rank.
- Rankable means: records no token (no market cap, no CoinGecko id, no symbol) and none
  of the watchlist rule's exclusions — no CEX, no Bridge / Canonical Bridge / Liquid
  Staking / Liquid Restaking / Restaked BTC category, no versioned name (trailing V2/v2/
  2.0), no whole word wrapped / staked / staking, no exchange-name prefix.
- The 20 rankable protocols above the floor that the house's watchlist leaves out (its
  rule also uses `parentProtocol`, which the snapshot's ten fields do not carry) are NOT
  ranked: they are mostly sub-products and pools of other protocols, and the watchlist
  is the floor the house drew.
- Live-token protocols are set aside in their own view at their market caps, never mixed
  into the ranked board.

## Farm plans

Six steps per protocol, each with a cost and a time, derived from the protocol's own
record (category, chains, TVL, age): a fresh-wallet safety step, a category-specific
main action, a weekly qualifying action, an age-derived hold period, a claim check, and
a farm log. Capital tiers come from TVL ($250 / $150 / $100 / $50); the hold is 60 days
for protocols under 180 days old, 45 under a year, 30 otherwise.

## The dashboard

A static page with no network: a sortable board (rank, protocol, TVL, 7d growth, age,
chains, implied value), search + category + chain filters, a Pre-token / Live token
tab, a detail view for every protocol (model breakdown step by step, with the page
recomputing the implied value from the figures and comparing it to the board's number),
and the model panel with every constant. Dark console theme, amber accent.

## How to run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input ../input/snapshot.json
```

Requires Node 22.18 or later and the three screenshots in `screenshots/` (board, detail,
filtered). The program writes `board.json`, `data.js`, `dashboard/`, `screenshots/` and
`dossier.json` into the output directory, and refuses a board the gate would refuse.

## Files

- `main.ts` — the program: parse the snapshot, build the board, write the outputs.
- `lib/model.ts` — the radar-2 model, the farm-plan generator, the board builder, the
  gate self-check (`boardIssue`), and the `data.js` writer.
- `vendor/makefun-board.ts` — the make.fun dossier library, copied unchanged.
- `dashboard/` — the page: `index.html`, `app.css`, `app.js` (38,186 bytes in all).
- `screenshots/` — the three screenshots (board, detail, filtered), captured from the
  built dashboard.
