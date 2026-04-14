# Lucky Games — Dev Spec

> Integration spec for bringing Flappy Luck (and future mini-games) into the Lucky Arena Telegram bot as a paid, leaderboard-driven weekly competition mini app.
>
> **Audience:** devs working on `lucky-multichain` (Python bot + admin bot) and `the-mini-app-of-all-mini-apps-multichain` (Next.js) and this repo (Flappy-Luck).
>
> **Status:** DRAFT — awaiting sign-off on the branded-game onboarding mechanism (see §8).

---

## 1. High-level overview

- Players tap a new **🎮 Play Games** button in the Lucky Arena bot → opens a **Games mini app** (separate BotFather app from the existing Privy wallet mini app — Telegram supports multiple mini apps per bot).
- Inside the Games mini app: a grid of active games. Each game has its own page with entry price, current pot, leaderboard, and **Play** button.
- Paying the entry price:
  - Debits player's Privy wallet (same flow as ticket purchases).
  - Mints **1 chip** to the Lucky chip jackpot (matches ticket behavior — see `src/services/chips.py:34`).
  - Fires the referral chip cascade (T1 + T2) using the entry-price USD equivalent, same as ticket purchases.
  - Creates a play session — player can immediately play once.
- Player can replay as many times as they want (each play = new payment = new entry = new chip + new referral cascade).
- Weekly (or admin-set duration): the top scorer per game wins the pot.
- **Ties split the winner-share evenly** (2 tie = each gets 35% (half of 70), 3 tie = ~23.3% each, etc.).
- After round ends, game enters `paused` state. **Admin must manually start the next round.**

---

## 2. Economics

### 2.1 Entry flow
- Entry price is set per-game-per-chain by admin (`SOL`, `BNB`, `BASE` — follows existing multichain pattern).
- Payment is debited from the player's Privy wallet via the same `sign_and_send_transaction` / `sign_and_send_evm_transaction` used for tickets (`src/services/privy.py:612` Solana, `:707` EVM).

### 2.2 Pot accumulation
- Each entry fee is split immediately in the DB accounting layer:
  - **28%** → Lucky house
  - **2%** → Chip jackpot pot
  - **70%** → Game pot for that round (the "winner pot")
- On-chain the full entry goes to the collections wallet; splits are bookkeeping. Payout to winner happens at round end from the collections wallet.

### 2.3 Tie handling
- Winner pot (70%) is split evenly across all players tied for #1.
- `payout_per_winner = floor(winner_pot / tie_count)` in smallest unit (lamports / wei).
- Any dust from flooring rolls to the next round's pot.

### 2.4 Chips & referrals
- **1 entry = 1 chip** awarded to the player (same ratio as tickets).
- Referral cascade fires on each entry:
  - T1 referrer gets 2 chips per $1 of entry-price USD
  - T2 referrer gets 1 chip per $1 of entry-price USD
- Reuse `src/services/referrals.py:47` `award_referral_chips()` — pass entry USD as the spend amount.

---

## 3. Rounds & scheduling

### 3.1 Round lifecycle
```
uploaded → configuring → ready → active → ended → paid → paused
                                      ↑                        |
                                      └──── admin starts ──────┘
```

1. **uploaded** — a new game slug has been detected from git (see §8). No config yet. Shows in admin panel as "New game — needs configuration".
2. **configuring** — admin is setting entry price, duration, chain availability.
3. **ready** — fully configured but not running. Admin needs to explicitly press "Start round".
4. **active** — round is live, players can enter, scores are being recorded.
5. **ended** — round duration expired. Scheduler picks winner(s), freezes leaderboard.
6. **paid** — payout tx confirmed on-chain for all tied winners per chain.
7. **paused** — default resting state between rounds. Admin must press "Start round" again to enter `active`.

### 3.2 Admin-gated starts
- No round ever auto-starts. Even on first configuration, `ready` → `active` requires admin action.
- Prevents white-labeled games from silently running empty rounds and paying out refunds-to-house.

### 3.3 Duration
- Admin-set per round. Typical: 7 days. Stored as seconds. Round end timestamp = `started_at + duration_seconds`.
- Scheduler loop (reuse `ChipJackpotScheduler` pattern in `src/services/chips.py:265`) checks every minute for `active` rounds past their end timestamp and transitions them to `ended`.

---

## 4. Changes in `lucky-multichain` (Python bot)

### 4.1 New DB tables
Add to `src/database/connection.py`:

```sql
-- Registered games (one row per distinct game instance, including branded variants)
CREATE TABLE games (
  id              SERIAL PRIMARY KEY,
  slug            VARCHAR(64) UNIQUE NOT NULL,        -- e.g. "flappy-luck", "flappy-luck-nala"
  name            VARCHAR(128) NOT NULL,              -- display name
  game_url        TEXT NOT NULL,                      -- Vercel deploy URL for the game HTML
  branding_json   JSONB,                              -- logo, colors, title, etc.
  is_enabled      BOOLEAN DEFAULT FALSE,              -- admin kill-switch
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  removed_at      TIMESTAMPTZ                         -- soft-delete for post-round cleanup
);

-- Rounds (one row per round per game per chain)
CREATE TABLE game_rounds (
  id              SERIAL PRIMARY KEY,
  game_id         INT REFERENCES games(id),
  chain           VARCHAR(16),                        -- 'solana' | 'bnb' | 'base'
  state           VARCHAR(16),                        -- see §3.1
  entry_price     NUMERIC(30, 9),                     -- in native token smallest unit
  duration_seconds INT,
  started_at      TIMESTAMPTZ,
  ends_at         TIMESTAMPTZ,
  current_pot     NUMERIC(30, 9) DEFAULT 0,           -- winner-share accumulator (60%)
  entries_count   INT DEFAULT 0,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_rounds_state ON game_rounds(state);
CREATE INDEX idx_rounds_active ON game_rounds(game_id, chain, state) WHERE state = 'active';

-- Entries (each paid play)
CREATE TABLE game_entries (
  id              SERIAL PRIMARY KEY,
  round_id        INT REFERENCES game_rounds(id),
  user_id         INT REFERENCES users(id),
  amount_paid     NUMERIC(30, 9),
  tx_signature    VARCHAR(128),
  chips_minted    INT DEFAULT 1,
  session_token   VARCHAR(256) UNIQUE,                -- short-lived JWT-ish token
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_entries_user ON game_entries(user_id, round_id);

-- Scores (submitted results)
CREATE TABLE game_scores (
  id              SERIAL PRIMARY KEY,
  entry_id        INT REFERENCES game_entries(id),
  round_id        INT REFERENCES game_rounds(id),
  user_id         INT REFERENCES users(id),
  score           INT,
  played_seconds  INT,                                -- time alive in the game
  replay_data     JSONB,                              -- for anti-bot re-simulation (§7)
  flagged         BOOLEAN DEFAULT FALSE,              -- anti-bot shadow flag
  submitted_at    TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_scores_leaderboard ON game_scores(round_id, score DESC, submitted_at)
  WHERE flagged = FALSE;

-- Payouts
CREATE TABLE game_payouts (
  id              SERIAL PRIMARY KEY,
  round_id        INT REFERENCES game_rounds(id),
  user_id         INT REFERENCES users(id),
  amount          NUMERIC(30, 9),
  tx_signature    VARCHAR(128),
  paid_at         TIMESTAMPTZ DEFAULT NOW()
);
```

### 4.2 New services
- `src/services/games.py`
  - `list_active_games(chain) → [game_summary]`
  - `get_lobby(slug, chain) → {entry_price, pot, leaderboard, round_state, ends_at}`
  - `create_entry(user_id, slug, chain) → session_token` — verifies wallet balance, calls Privy transfer, creates `game_entries` row, awards chip, triggers referral cascade. **Transactional.**
  - `submit_score(session_token, score, replay_data) → {accepted|rejected, rank}` — validates session, anti-bot checks (§7), inserts `game_scores`.
  - `get_leaderboard(slug, chain, round_id=current) → [(wallet, score)]`

- `src/services/game_scheduler.py`
  - Background loop (mirrors `ChipJackpotScheduler._run_scheduler` pattern).
  - Every 60s: find `active` rounds with `ends_at <= now()` → transition to `ended` → pick winners → pay → transition to `paid` → eventually `paused`.
  - Winner selection: top score in `game_scores` for that round where `flagged = FALSE`, tied = split.

- `src/services/game_registry.py`
  - Auto-discovery of new games (§8).
  - `sync_manifest()` → reads manifest → inserts new `games` rows with `is_enabled = FALSE`, state = `uploaded`.
  - Called on bot startup + hourly interval.

### 4.3 New handlers (user bot)
- `src/handlers/games.py`
  - `/play` or "🎮 Play Games" button → sends `WebAppInfo` button with `GAMES_MINIAPP_URL` (new env var, separate from Privy miniapp URL).
  - No other user-bot interaction — the miniapp owns the game UX end-to-end.

### 4.4 New handlers (admin bot)
- `src/admin_handlers/games.py`
  - `/admin games` → lists all games with state badges:
    - 🆕 Uploaded (needs configuration)
    - ⚙️ Configuring
    - 🟢 Active (round live)
    - 🟡 Paused (ready to start)
    - 🔴 Disabled
  - Per-game submenu:
    - **Set entry price** (per chain) — reuses pattern from `src/admin_handlers/draws.py:147`
    - **Set duration** (seconds)
    - **Enable / Disable** game entirely
    - **Start round** — transitions `paused → active`, stamps `started_at`, computes `ends_at`
    - **Force-end round** — early stop
    - **View current round** — shows pot, entry count, top 10 leaderboard
    - **View payout history**
    - **Remove game** — soft-delete (`removed_at = NOW()`) after final round paid out

### 4.5 Environment / config additions
- `GAMES_MINIAPP_URL` — base URL of the games mini app Vercel deployment.
- `GAMES_MANIFEST_URL` — URL or repo path where the games manifest lives (§8).
- `GAME_SESSION_SECRET` — HMAC key for signing session tokens.

### 4.6 What to reuse (DO NOT rebuild)
| Need | Existing |
|---|---|
| initData HMAC verification | `src/lib/auth.ts` (miniapp) |
| Privy wallet debit | `src/services/privy.py:612, :707` |
| Admin settings storage | `SettingsQueries` |
| Admin prompt/callback pattern | `src/admin_handlers/draws.py:147` |
| Chip minting | `src/services/chips.py:34` |
| Referral cascade | `src/services/referrals.py:47` |
| Weekly scheduler loop | `src/services/chips.py:265` |

---

## 5. Changes in `the-mini-app-of-all-mini-apps-multichain`

> **Option being pursued:** all games live inside one new Games mini app. Existing Privy wallet mini app stays untouched. This could be a new Next.js project OR new route tree under the existing one — either works.

Assuming a new top-level route tree `/games/*`:

### 5.1 Routes
- `/games` — grid of active games (fetches `games.list`)
- `/games/[slug]` — lobby: pot, entry price, leaderboard, Play button
- `/games/[slug]/play` — host page that loads the game HTML in an iframe, relays messages

### 5.2 API routes
All verify Telegram initData via existing `src/lib/auth.ts`:
- `POST /api/games/list` → active games + metadata
- `POST /api/games/lobby` → body: `{slug, chain}` → lobby state
- `POST /api/games/play` → body: `{slug, chain}` → debits wallet, returns `session_token`
- `POST /api/games/score` → body: `{session_token, score, replay}` → validated + stored
- `POST /api/games/leaderboard` → body: `{slug, chain, round_id?}` → paginated

### 5.3 Game host page
`/games/[slug]/play` loads an iframe pointing at the game's `game_url` (from `games.game_url`) with query params:
```
?session={session_token}&brand={brand_slug}&theme={tg-theme-json}
```
Listens for `postMessage` events from the iframe:
- `{type: 'ready'}` — game loaded
- `{type: 'score', score, replay}` — on game over → calls `/api/games/score`
- `{type: 'haptic', kind}` — optional relay to `Telegram.WebApp.HapticFeedback`

### 5.4 Leaderboard UI
- Wallet addresses truncated (`0xAbC…d3F` for EVM, `AbCd…xyZ1` for Solana).
- Current user row highlighted.
- Show rank + score + chain badge.
- Update in near-real-time (poll every 10s while on lobby page).

---

## 6. Changes in this repo (Flappy-Luck)

The game itself needs to become **embed-ready** and **branding-capable**. Keep it a single HTML file (no build step) — this is a feature not a bug.

### 6.1 CAPTCHA wrapper
Before the game canvas loads, the host page (`/games/[slug]/play`) shows the CAPTCHA widget. Only after success is `session_token` issued and the iframe loaded. See §7.3 for trigger policy.

### 6.2 Telegram SDK integration
```html
<script src="https://telegram.org/js/telegram-web-app.js"></script>
```
- Call `Telegram.WebApp.ready()` and `Telegram.WebApp.expand()` on load.
- Use `Telegram.WebApp.viewportHeight` in the `resize()` function instead of `window.innerHeight`.
- Pull theme vars (`--tg-theme-bg-color`, `--tg-theme-text-color`) with sensible dark-mode fallbacks.
- Fire haptics on flap (`light`) / score (`light`) / death (`heavy`).
- Cloud storage for personal best: `Telegram.WebApp.CloudStorage.setItem('pb_flappy', ...)`.

### 6.3 Session + branding query params
Parse on load:
- `session` — bearer token, echoed back on score submission
- `brand` — slug; selects `brandConfig[brand]` for theming
- `api` — base URL to post scores to (or use `postMessage` to parent — see §6.4)

### 6.4 Branding hooks
Pull a small config object keyed by `brand`:
```js
const BRANDS = {
  default: { bg: '#1a1a2e', bird: '#fbbf24', pipe: '#7c3aed', title: 'Flappy Luck' },
  nala:    { bg: '#0f172a', bird: '#22c55e', pipe: '#ef4444', title: 'Flappy Nala' },
  // ... added per branded branch
};
```
Branded branches override `BRANDS.default` and/or add their own entries. Changing colors + title + maybe a sprite swap = rebrand.

### 6.5 Score submission
On game over, either:
- **`postMessage` to parent** (preferred — parent page owns the API call with `session_token`):
  ```js
  parent.postMessage({ type: 'score', score, replay }, '*');
  ```
- Or direct `fetch(api + '/api/games/score', {...})` if embedded standalone.

### 6.6 Replay recording (for anti-bot)
Record a compressed array of input events during play:
```js
const replay = { seed, events: [[ts, 'flap'], [ts, 'flap'], ...] };
```
Include `replay` in the score submission payload. Server re-simulates to verify (see §7).

### 6.7 Input rate sanity
Reject inputs arriving faster than ~30ms apart (catches naive auto-clickers at the client — not a security layer, just friction).

---

## 7. Anti-tampering & anti-bot

> Two distinct threats with distinct defenses. A single defense doesn't cover both.

### 7.1 Threat model

| # | Threat | Scenario | Why it matters |
|---|---|---|---|
| **A** | **Client score tampering** | Human player modifies the JS, pays 1 entry, submits `{score: 99999}`, wins the pot with one play | A single attacker ends the round instantly. Other players stop playing because they know they can't win. |
| **B** | **Bot farming** | Attacker runs automated headless clients that actually play the game; plays hundreds of legitimate entries targeting mid-high scores | Bots beat casual players on consistency. Humans feel they can't win and disengage. |

Both threats kill the "feeling of a reason to play" that the user called out. Revenue-positive in the short term (bots pay entry fees), revenue-negative long-term (humans leave).

### 7.2 Defense A — server-side replay validation (stops score tampering)

**Primary defense against Threat A.**

- Client records every input event during a play, plus the RNG seed used for pipe spawns:
  ```js
  replay = { seed: 12345, events: [[ts_ms, 'flap'], [ts_ms, 'flap'], ...] }
  ```
- On game over, client submits `{score, replay}`.
- Server re-runs the Flappy physics simulation deterministically using the same seed + same event stream.
- Server accepts the score only if the simulated score **exactly matches** the submitted score. (Flappy physics is fully deterministic — constants in `index.html`: `GRAV = 0.2`, `FLAP = -4.5`, `PSPD = 2`, `GAP = 200`, `PW = 45`, `SPAWN = 200`, `BSIZE = 15`, `GH = 60`.)
- Tampered scores fail this check because the attacker can't produce a valid `(seed, events)` pair that simulates to their forged score — they'd have to actually play to score 99999, which defeats the point.
- This also catches the most common cheat vector with zero UX cost to honest players.

**Implementation note:** put the physics simulator in a shared JS module; use it both client-side (in the game) and server-side (in a Node runtime inside the Next.js API route, or exported to a pure-JS worker). Shared code = no drift.

### 7.3 Defense B — CAPTCHA + behavioral signals (stops bot farming)

**Primary defense against Threat B.**

- **CAPTCHA on session creation** — before issuing a `session_token` (i.e. before the game canvas loads), require the player to pass a Cloudflare Turnstile or hCaptcha challenge.
  - Trigger policy options, pick one:
    - **Always** — every play requires a fresh CAPTCHA. Highest friction, strongest defense.
    - **First play per session** — CAPTCHA once per 30-min window; subsequent plays in that window skip it. Balanced.
    - **Adaptive** — only trigger CAPTCHA on suspicion (after N rapid plays, or if prior replay looked suspicious). Lowest friction, moderate defense. Requires building the suspicion scorer.
  - **Recommend: "First play per session"** for v1 — acceptable UX, blocks naive bots, extensible to adaptive later.
- **Behavioral replay analysis** (runs on every replay, even after CAPTCHA):
  - Flap-interval standard deviation — humans σ > threshold, metronome bots σ ≈ 0.
  - Reaction-time distribution — humans show a bell curve around ~250ms, bots show Dirac deltas or uniform.
  - If replay statistics look non-human: shadow-flag the score (`game_scores.flagged = TRUE` → excluded from leaderboard, no feedback to the player that they were flagged). Shadow-flagging beats hard-reject because bot authors can't tell what triggered detection.

### 7.4 Defense C — rate limits (defense in depth)

- Per-Telegram-ID: max 1 concurrent active session (no two plays in flight at once).
- Per-Telegram-ID: max 10 plays per minute (generous for humans, annoying for bots).
- Per-Telegram-ID: max 500 plays per round (hard cap — at that point the player is either a whale or a bot that passed CAPTCHA somehow; admin should review).
- Enforce at `create_entry()` in `src/services/games.py`.

### 7.5 Economic ceiling
- Entry fees are real money, so `E[attacker profit] = P(win) × winner_share − N × entry_price`.
- With replay validation + CAPTCHA, `P(win)` for a bot approaches the share it would earn playing honestly. Payoff converges to zero.
- With no defenses, `P(win) ≈ 1` and the attacker wins ~70% of every entry fee ever collected. Gap is huge — the defenses pay for themselves.

### 7.6 What we explicitly DO NOT do
- **Don't try to block "honest cheating"** where a player pads their own score without tampering (e.g. pays a human to play for them). User flagged this as revenue-positive and out of scope.
- **Don't device-fingerprint** — Telegram mini app sandbox limits this and honest users benefit from privacy.
- **Don't hard-reject flagged scores silently** — always shadow-flag so the attacker gets no debug signal.

---

## 8. Branded games — onboarding mechanism

> User constraint: branded games come from git, not the admin panel. Branch or copy of Flappy-Luck → rebrand → deploy → appears in admin panel → admin configures + starts.

Two options — **pick one before implementation**:

### Option A — `games-registry.json` manifest in a known repo (RECOMMENDED)
1. Create a new repo `lucky-games-registry` with a single `registry.json`:
   ```json
   [
     {
       "slug": "flappy-luck",
       "name": "Flappy Luck",
       "game_url": "https://flappy-luck.vercel.app",
       "branding": { "title": "Flappy Luck" }
     },
     {
       "slug": "flappy-luck-nala",
       "name": "Flappy Nala",
       "game_url": "https://flappy-luck-nala.vercel.app",
       "branding": { "title": "Flappy Nala", "bird": "#22c55e" }
     }
   ]
   ```
2. Deploying a branded game = push a branded branch → Vercel auto-deploys → dev PRs a new entry into `registry.json`.
3. Bot's `game_registry.sync_manifest()` fetches the raw GitHub URL of `registry.json` on startup + hourly.
4. New entries → `INSERT INTO games (slug, name, game_url, branding_json, is_enabled) VALUES (..., FALSE)`. Removed entries → `UPDATE games SET is_enabled = FALSE WHERE slug = ...`.
5. Admin sees new game in `/admin games` panel with 🆕 badge.
6. **Pros:** simple, explicit, auditable via git history, no webhooks.
7. **Cons:** requires a PR to the registry repo each time.

### Option B — GitHub branch-scan via API
1. Bot hits `GET /repos/NalaDev33/Flappy-Luck/branches` on interval.
2. Any branch matching `flappy-luck-*` pattern is treated as a branded variant.
3. Assumes a convention: branch name = slug, Vercel preview URL = `https://flappy-luck-git-{branch}-{team}.vercel.app` (or configured pattern).
4. **Pros:** truly zero-touch — just push a branch.
5. **Cons:** fragile (URL patterns drift), harder to pass per-brand metadata, deleted branches silently disable games.

**Recommendation: Option A.** Explicit > magic when real money is moving.

### 8.1 Removing a game after a round
- Admin calls **Remove game** in panel → soft-delete (`removed_at = NOW()`, `is_enabled = FALSE`).
- Does NOT delete historical `game_entries`, `game_scores`, `game_payouts`, or `game_rounds` — preserves accounting.
- Registry can then remove the entry from `registry.json` and delete the Vercel deploy.

---

## 9. API contracts (detail)

All endpoints served from the mini app's Next.js API routes. All verify `initData` first.

### `POST /api/games/list`
```json
Request:  { "initData": "...", "chain": "solana" }
Response: {
  "games": [
    {
      "slug": "flappy-luck",
      "name": "Flappy Luck",
      "branding": {...},
      "round": {
        "state": "active",
        "ends_at": "2026-04-21T20:00:00Z",
        "entry_price_native": "0.05",
        "entry_price_usd": "7.50",
        "pot_usd": "142.00",
        "entries_count": 38
      }
    }
  ]
}
```

### `POST /api/games/play`
```json
Request:  { "initData": "...", "slug": "flappy-luck", "chain": "solana" }
Response: {
  "session_token": "eyJ...",       // HMAC-signed, 15-min TTL, bound to entry_id
  "entry_id": 12345,
  "tx_signature": "5kG..."
}
Errors:   insufficient_balance, round_not_active, game_disabled, rate_limited
```

### `POST /api/games/score`
```json
Request:  { "initData": "...", "session_token": "eyJ...", "score": 42,
            "played_seconds": 87, "replay": { "seed": 12345, "events": [[t,"flap"],...] } }
Response: { "accepted": true, "rank": 7 }     // or { "accepted": false, "reason": "replay_mismatch" }
```

### `POST /api/games/leaderboard`
```json
Request:  { "initData": "...", "slug": "flappy-luck", "chain": "solana", "limit": 50 }
Response: {
  "round_id": 17,
  "ends_at": "2026-04-21T20:00:00Z",
  "current_user_rank": 12,
  "entries": [
    { "rank": 1, "wallet": "AbCd…xyZ1", "score": 142, "played_at": "..." },
    ...
  ]
}
```

---

## 10. Open questions (blocking)

1. **Branded-game mechanism** — Option A (manifest) or Option B (branch scan). See §8.
2. **CAPTCHA trigger policy** — "always", "first play per session", or "adaptive"? See §7.3. Recommend "first play per session".
3. **Min entrants threshold** — if only 1 person enters, do they auto-win or does the round void and refund? Current spec: they win (simplest).
4. **Leaderboard refresh frequency** — client polls every 10s (default). Acceptable?
5. **Replay payload size cap** — a long game generates many events. Hard-cap replay to e.g. 10KB compressed → force game-over at equivalent score if exceeded.

See also: [CHEATING.md](./CHEATING.md) for the full threat catalog.

---

## 11. Delivery order (suggested)

1. Anti-bot replay validator spike (prove the approach works) — 1 day
2. DB migrations + `games` + `game_registry` scaffolding — 1 day
3. Admin bot `/admin games` menu — 2 days
4. User bot "🎮 Play Games" button + env wiring — 0.5 day
5. Games mini app `/games` + `/games/[slug]` pages + API routes — 3 days
6. `games/[slug]/play` host page + postMessage wiring — 1 day
7. Flappy-Luck SDK integration + replay recording + branding — 1.5 days
8. Scheduler + payout logic + tie-handling — 2 days
9. End-to-end QA on testnet with 3 fake users — 1 day

**Total rough estimate: ~13 days of focused dev work.**

---

## 12. Not in scope (v1)

- Profile / stats pages (stats visible only via leaderboard rank)
- Cross-game meta-leaderboard ("best player across all games")
- Tournaments / bracketed competition
- Social share cards for score
- Practice mode / free plays
- Mobile push notifications for "round ending soon" / "you've been overtaken"

These are all reasonable v2 candidates once the core loop proves out.
