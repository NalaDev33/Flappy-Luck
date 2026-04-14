# Lucky Games — Cheating & Abuse Catalog

> Full enumeration of cheating vectors for Flappy Luck (and future games), with feasibility, mitigation, and residual risk per vector.
>
> **Companion doc to:** [DEV_SPEC.md](./DEV_SPEC.md) (§7 covers the high-level defense model).
>
> **Principle:** we cannot prevent all cheating. We can make cheating cost more than it pays, and we can prevent the specific cheats that would demoralize honest players and kill the game. The goal is *humans feel they have a real chance to win*.

---

## Why scores can be forged in the first place

The game runs in the player's browser as JavaScript. That code is **on the player's machine, under their control**. Everything the game does — scoring, collision detection, physics, submitting the result — happens in an environment the player can inspect and modify.

Ways a player can trivially produce a fake score, with zero tools beyond a browser:

1. **Browser DevTools console.** Open F12 (or long-press on mobile with remote debugging), find the submission function, call it with any score:
   ```js
   fetch('/api/games/score', {
     method: 'POST',
     body: JSON.stringify({ session_token: '...', score: 999999 })
   });
   ```
2. **Edit the JS before it runs.** Use a browser extension like Tampermonkey, or save the page and load a modified copy, to change the game so that every pipe passed increments the score by 1000 instead of 1. Then play one pipe, submit, win.
3. **Proxy / intercept.** Run the real game, let it submit a real score of 7, intercept the HTTP request in mitmproxy, change the body to `score: 99999`, forward it.
4. **Directly call the server API.** Skip the game entirely. Get a `session_token` (by legitimately paying to play), then just POST a fake score without ever loading the canvas.
5. **Modify the replay events.** If we trust a client-submitted `(score, events)` pair without re-running the simulation, player can hand-craft an events array that "explains" any score.

All of these attacks are within reach of a semi-technical player. A Discord tutorial would democratize them. **The defense has to assume the client is fully adversarial.**

---

## Threat catalog

Each threat lists:
- **ID** — short code for reference in code comments / tickets
- **Feasibility** — how easy is this attack? (`trivial`, `easy`, `medium`, `hard`, `research-grade`)
- **Mitigation** — what we do
- **Residual risk** — what's still possible after mitigation

---

### T1 — Direct score injection

**Attack:** Player pays for entry, opens DevTools, calls the score API with `{score: 99999}` without playing.

**Feasibility:** trivial.

**Mitigation:**
- Server requires `replay` payload alongside `score`.
- Server re-simulates the game deterministically using the replay's `seed` and `events`.
- Accepts only if simulated score === submitted score.
- Without a valid replay, any score is rejected.

**Residual risk:** none for this specific vector. Attacker must produce a valid replay → see T2 and T3.

---

### T2 — Replay forgery (synthetic events)

**Attack:** Player understands Flappy physics, hand-crafts an `events` array (flap timestamps) that they believe will produce a high score, submits it with that score.

**Feasibility:** medium. Requires reverse-engineering the physics. Once one person figures it out, scripts can be shared.

**Mitigation:**
- The server runs the *canonical* simulation. If the attacker's predicted score doesn't match what the simulator produces from their events, submission rejected.
- So the attacker has to actually construct a valid input sequence that scores high. This is equivalent to "write a bot that plays Flappy well," which is T7 territory — it's a real engineering project, not a cheat.

**Residual risk:** a dedicated attacker can write a Flappy-solving script that produces valid high-scoring replays. Caught by T7 defenses (CAPTCHA + behavioral).

---

### T3 — Replay re-use (record-and-replay)

**Attack:** Attacker plays once legitimately, scores well, records the `(seed, events)` pair, then re-submits the same replay with a new session_token from subsequent paid entries — paying for entries but never actually playing, just re-submitting the same winning replay.

**Feasibility:** easy if allowed.

**Mitigation:**
- Each entry has a unique `seed` generated **server-side** when `session_token` is issued.
- The seed controls pipe-gap positions. A replay from seed A won't simulate correctly against seed B — the pipes are in different places, so the same flap sequence crashes into a pipe.
- Even if attacker somehow reuses the same seed across entries, `session_token` is single-use (DB unique constraint on `game_scores.session_token` or similar one-score-per-entry rule).

**Residual risk:** none.

---

### T4 — Session token theft

**Attack:** Attacker steals another user's `session_token` (via XSS, shared device, MITM) and submits scores under their entry.

**Feasibility:** hard. Telegram mini app runs in a sandboxed webview; HTTPS-only; no obvious XSS surface in the game itself.

**Mitigation:**
- `session_token` is bound server-side to the originating Telegram user ID at issue time.
- Score submission re-verifies `initData` in the same request — the Telegram user submitting must match the user the token was issued to.
- Token TTL: 15 minutes. Expired tokens rejected.

**Residual risk:** extremely low. Requires both stealing the token AND forging Telegram initData (which requires the bot's secret — game-over if that leaks, but unrelated to this feature).

---

### T5 — Race condition / double-submit

**Attack:** Attacker submits the same session_token twice in parallel, hoping one succeeds while the other also gets credited.

**Feasibility:** trivial.

**Mitigation:**
- DB unique constraint: `UNIQUE (session_token)` on `game_scores`.
- First submission wins; second errors at insert time.
- Enforced at the storage layer — not reliant on app-layer logic.

**Residual risk:** none.

---

### T6 — Oversized replay payload (DoS / resource exhaustion)

**Attack:** Submit a replay with millions of events, or malformed JSON, to DoS the server or crash the simulator.

**Feasibility:** trivial.

**Mitigation:**
- Hard cap on request body size at the API layer (e.g. 16KB).
- Hard cap on `events.length` at parse time (e.g. 5000 events).
- Simulator has a max-tick ceiling (e.g. 30 minutes of game time) — exits early if exceeded.
- Malformed JSON rejected at parse.

**Residual risk:** none for this vector.

---

### T7 — Bot farming (automated real play)

**Attack:** Attacker writes an automated client that actually plays the game — either a scripted bot (rule-based flapping), a computer-vision-based reaction bot, or an ML policy trained to play Flappy. Each play is a real, valid replay that simulates correctly. The bot just plays many times and scores well.

**Feasibility:** medium (scripted) to research-grade (ML). Someone will try it.

**Mitigation stack** (layered, each adds friction):

1. **CAPTCHA on session creation** — Cloudflare Turnstile or hCaptcha before `session_token` is issued. Trigger policy TBD (recommend "first play per 30-min session"). Stops fully-automated bots that can't solve CAPTCHAs. Does NOT stop CAPTCHA farms where humans pre-solve CAPTCHAs for bots — but those add cost and latency per play.
2. **Behavioral replay analysis** — even if the bot gets a valid session, the replay's flap intervals reveal the bot:
   - Flap-interval standard deviation: humans ~50-150ms, metronome bots ~0.
   - Flap-count-per-pipe consistency: humans vary, rule-based bots don't.
   - Reaction-time distribution (time from pipe-entering-danger-zone to flap): humans show bell curves; bots show deltas.
   - Score the replay; if anomaly score > threshold, **shadow-flag** (`game_scores.flagged = TRUE`). Flagged scores are excluded from leaderboard but still recorded for audit. Player receives no feedback — bot author can't tell what's detected.
3. **Rate limits** — per-user play caps (see DEV_SPEC §7.4). Limits the blast radius of a single bot account.
4. **Payment as economic cost** — every play costs money. Expected profit `= P(win) × 70% pot − N × entry_price`. If the bot can't reliably beat humans, profit goes negative. CAPTCHA cost per play amplifies this.

**Residual risk:** a sophisticated attacker running an ML-trained bot that mimics human input distributions, behind human-solved CAPTCHAs, will beat these defenses. At that point they've invested engineering effort comparable to a small research project. Weigh against expected winnings. For v1 this is acceptable — revisit if abuse materializes.

---

### T8 — Shared account / ghost player

**Attack:** Skilled Flappy player is paid (or volunteers) to play on someone else's Telegram account.

**Feasibility:** trivial socially.

**Mitigation:** none. This is indistinguishable from the legitimate account holder simply being good.

**Why we don't care:** user explicitly flagged this as revenue-positive and out of scope. Treat it as a human playing, because it *is* a human playing.

---

### T9 — Multi-account sybil farming

**Attack:** One person controls 20 Telegram accounts, enters each in every round to dominate the top of the leaderboard.

**Feasibility:** easy but bounded by capital — each entry costs real money.

**Mitigation:**
- No anti-sybil enforcement at account creation (Telegram handles its own).
- Economic: every account's entries are real revenue. The attacker pays us N× the entry fee for their N accounts; they can only win one pot.
- Expected profit math is the same as T7 — breaks even at best unless they're actually better than the field.

**Residual risk:** sybil farming a branded game with low entrants can work (small pot, few competitors). Branded-game admin discretion on `start round` mitigates this — don't start rounds you don't expect to attract real players.

---

### T10 — Speedhacks / time dilation

**Attack:** Player slows down their client's clock (DevTools throttle, modify `requestAnimationFrame`) so pipes move past in slow motion, making the game trivially easy.

**Feasibility:** easy with DevTools.

**Mitigation:**
- **Replay is a logical event stream, not wall-clock timing.** The server re-simulates using the *game's internal tick counter*, not real-world time.
- The events array records `[frame_number, 'flap']` (or an equivalent game-tick counter) rather than `[real_wall_time_ms, 'flap']`. Server simulates frame-by-frame deterministically.
- Speedhacks change how fast the player perceives the game, but the resulting replay is indistinguishable from a normal-speed play — and the replay still has to pass through the server simulator.
- **However:** if we accidentally use wall-clock timestamps in the replay, this attack becomes devastating — the player can play at 0.1× speed and the server can't tell. **Implementation note for devs: events must use game ticks, not ms timestamps.**

**Residual risk:** if implementation slips on the ticks-vs-timestamps rule, this threat is wide-open. Call this out in PR review.

---

### T11 — Physics constant tampering

**Attack:** Player edits client JS to change gravity from 0.2 to 0.01 so the bird barely falls; plays; submits replay.

**Feasibility:** trivial.

**Mitigation:**
- Server simulator uses **canonical physics constants** from its own code, not anything the client sends.
- A replay recorded against modified client physics won't simulate to the same score against server-side canonical physics — the bird would drop faster server-side, flap events would be off, score wouldn't match → rejected.

**Residual risk:** none, assuming the simulator's constants are frozen and match the canonical game build.

---

### T12 — Client pause-and-compute

**Attack:** Player scripts the game to pause every frame, read the bird+pipe positions, compute the optimal flap, unpause, repeat. Produces perfect replays at human-looking speeds if timed carefully.

**Feasibility:** medium — requires scripting knowledge; becomes T7 bot.

**Mitigation:** this is effectively a bot. Caught by behavioral analysis (T7.2) because the resulting flap timings will be suspiciously precise — hitting the optimal flap frame every time produces near-zero timing variance.

**Residual risk:** same as T7. Acceptable tradeoff.

---

### T13 — Leaderboard griefing (lowball submissions)

**Attack:** Player submits a low score hoping it somehow knocks out their previous high score, or tries to replace a high score with a low one to hide their performance.

**Feasibility:** trivial.

**Mitigation:**
- Leaderboard query uses `MAX(score)` per user per round, not "most recent score".
- Players can submit as many scores as they want; only the best counts.

**Residual risk:** none.

---

### T14 — Withdraw / refund timing games

**Attack:** Player enters, sees they're not winning, tries to withdraw entry fee before round ends.

**Feasibility:** depends on implementation.

**Mitigation:**
- Entry fees are **non-refundable** by design. Once `game_entries` row is created and tx_signature confirmed, no refund path exists.
- Admin can refund manually if needed (exceptional cases only) via a separate admin tool.

**Residual risk:** none.

---

### T15 — Admin / insider abuse

**Attack:** A person with admin bot access forces rounds, pays out to themselves, or fabricates scores.

**Feasibility:** depends on access.

**Mitigation:**
- All admin actions logged to an `admin_audit` table (timestamp, admin_telegram_id, action, params).
- Payout tx_signatures are on-chain and publicly verifiable.
- Admin bot access restricted to a known allowlist (existing pattern in `lucky-multichain`).

**Residual risk:** insider risk is outside the scope of this game feature; governed by org-level access control.

---

### T16 — Replay compression / format fuzz

**Attack:** Submit a replay that exploits a parser bug (prototype pollution, deserialization RCE, etc.) to escalate beyond the game's API surface.

**Feasibility:** low — requires finding a bug in the JSON parser or our code.

**Mitigation:**
- Strict schema validation on replay payloads (allowlist of event types, numeric range checks, length limits).
- Server simulator is pure JS — no eval, no dynamic code.
- Standard web app hardening (helmet, body-parser limits) applied at the Next.js API layer.

**Residual risk:** same as any web app — depends on code quality. Standard security review applies.

---

### T17 — CAPTCHA farm + ML bot (composite high-effort attack)

**Attack:** Combine CAPTCHA farm ($0.001/solve via 2Captcha-style services), ML-trained Flappy bot tuned to mimic human input distributions, and multiple Telegram accounts. Submits many valid, behaviorally-plausible replays.

**Feasibility:** research-grade.

**Mitigation:**
- Each entry fee pays us revenue.
- Adaptive CAPTCHA: if per-account play rate exceeds normal, escalate CAPTCHA frequency or difficulty.
- Admin monitoring: if a single account is dominating a branded game, admin reviews replays manually and can ban + claw back.
- Cross-account correlation: if N accounts using the same Privy wallet signing key or same IP hit range pattern, flag for review.

**Residual risk:** this is the "end-game" cheat. If someone invests the effort, they'll win sometimes. We detect patterns, restrict, and accept some loss. If losses exceed ~5% of prize pool we add harder measures.

---

## Summary table

| ID | Name | Feasibility | Defense status |
|----|------|-------------|----------------|
| T1 | Direct score injection | trivial | fully mitigated (replay validation) |
| T2 | Replay forgery (synthetic) | medium | degrades to T7 |
| T3 | Replay re-use | easy | fully mitigated (server-side seed + single-use token) |
| T4 | Session token theft | hard | fully mitigated (initData binding) |
| T5 | Race / double-submit | trivial | fully mitigated (DB unique constraint) |
| T6 | Oversized replay DoS | trivial | fully mitigated (size caps) |
| T7 | Bot farming | medium–research | mitigated (CAPTCHA + behavioral + rate + economics); residual accepted |
| T8 | Ghost player | trivial socially | out of scope by design |
| T9 | Sybil farming | easy | economically self-limiting |
| T10 | Speedhacks | easy | fully mitigated *IF* replays use game-ticks not wall-clock |
| T11 | Physics tampering | trivial | fully mitigated (server canonical constants) |
| T12 | Pause-and-compute | medium | degrades to T7 |
| T13 | Leaderboard griefing | trivial | fully mitigated (MAX-per-user) |
| T14 | Refund timing | n/a | non-refundable by design |
| T15 | Admin insider | access-dependent | audit log + on-chain payouts |
| T16 | Parser exploits | low | standard hardening |
| T17 | CAPTCHA farm + ML bot | research-grade | partially mitigated; monitored |

---

## Implementation checklist for devs

- [ ] Server-side deterministic physics simulator (authoritative, NOT shared with client)
- [ ] Replay payload schema + strict validator at API boundary
- [ ] Replays use **game-tick indices**, not wall-clock timestamps (see T10)
- [ ] `game_scores.session_token UNIQUE` constraint
- [ ] Server-issued `seed` per session (not client-provided)
- [ ] `session_token` bound to telegram_id at issue time; re-verified on submit
- [ ] Cloudflare Turnstile or hCaptcha integrated on `/games/[slug]/play`
- [ ] Behavioral scorer module (flap-interval σ, reaction-time shape)
- [ ] `game_scores.flagged` column; leaderboard query excludes flagged
- [ ] Rate limits in `create_entry()` (concurrent sessions, per-minute caps)
- [ ] Request body size limit at Next.js API layer (16KB)
- [ ] Admin audit log for all `/admin games` actions
- [ ] Canonical physics constants frozen in a `GAME_CONSTS` module imported by both game + simulator
- [ ] Runbook: how to review flagged scores, manually refund, and ban accounts

---

## Not in the catalog (intentionally)

- **Social engineering outside the app** (phishing, scam DMs, impersonation) — bot-level security concern, not game-specific.
- **Telegram bot token leakage** — catastrophic but unrelated to games.
- **Privy wallet compromise** — handled by Privy's own security model.
- **Smart contract / on-chain exploits** — payouts are simple transfers, no contract logic to exploit.
