# Engineering Log — Production Bugs, Root Causes & Fixes

This log documents the real debugging sessions that took the mirror system from
"entry works but the exit never happens" to reliable millisecond mirroring.
Each entry is a genuine production incident.

> No credentials, IP addresses, account identifiers, or tokens appear here.

---

## 1. Exits silently skipped — the "orphan-exit guard"

**Symptom:** The operator would enter when the retailer entered, but frequently
*stay in a position after the retailer had exited*.

**Root cause:** The original logic mirrored each order by flipping the side and
*guessing* whether the incoming order was an entry or an exit. A guard meant to
avoid opening a wrong-direction position would, in several cases, **skip the exit
leg entirely**, leaving the operator stranded.

**Fix:** Replaced order-by-order guessing with **position-target convergence**.
The master sends its *net position after each fill*; the operator computes
`target = -round(master_net × multiplier)` and places the single order that
closes the gap. Entries, partial exits, full exits, and reversals are now all
correct by construction, and any single missed event self-heals on the next one.

---

## 2. The safety-net reconciler was a no-op

**Symptom:** Even with a "last line of defence" reconciler running every 30 s,
missed exits were never caught.

**Root cause:** Two independent flaws:
1. It ran in **report-only mode** — it logged "drift detected" but never placed a
   corrective order.
2. It **could not see the master's positions at all** — it read a local snapshot
   file that only ever contained its *own* side, so every cycle hit a
   "snapshot was from my own side — skip" branch and reconciled nothing.

**Fix:** The master side now **pushes an HMAC-signed position snapshot** to the
follower every few seconds. The follower reconciler reads that fresh snapshot,
computes drift, and **force-corrects** — guarded by confirm-twice, a freshness
gate, a quantity cap, and the kill switch.

---

## 3. Runaway correction loop against manual hedges

**Symptom:** After enabling force-correct, the reconciler fired **70+ orders** in
a short window, targeting the operator's own manually-placed hedge legs.

**Root cause:** The operator holds deep-OTM, sub-₹5 options as *manual hedges* for
margin benefit. The master never holds these, so the reconciler saw them as
"drift" (operator has a position the master doesn't) and repeatedly tried to
"correct" (close) them.

**Fixes:**
- **Hedge protection by price.** Any position whose entry price is below a small
  threshold is treated as a manual hedge and excluded from reconciliation.
  (Product type can *not* be used as the discriminator — see incident #6.)
- **Circuit breaker.** If more than *N* corrections fire for one symbol — or more
  than *M* total — within a rolling window, the reconciler **trips the kill
  switch and stops**. An infinite loop is now structurally impossible.

**Damage control:** The emergency was stopped by engaging the kill switch within
seconds; most of the runaway orders were rejected by the broker, limiting the
impact to a single leg.

---

## 4. A regression I introduced — and how it was caught

**Symptom:** After a deploy, exits stopped mirroring again.

**Root cause:** A deploy shipped an **older local copy** of the follower that was
missing the convergence logic entirely — a self-inflicted regression.

**Fix:** Restored the correct convergence build from an on-box backup, re-applied
the newer changes on top, and added a **post-deploy verification** step that
confirms the expected logic is present (`grep` for the convergence markers) and
that services report healthy before the deploy is considered done.

**Lesson:** Always diff the artifact against what's actually running before
restarting a live money service; never assume the local copy is newest.

---

## 5. The order-update WebSocket was completely dead

**Symptom:** Entries arrived with a ~10 s delay; the fast path appeared not to
work at all.

**Root cause:** The broker SDK exposes both a *module* and a *class* with very
similar names. The code's "find the websocket class" loop matched the **module**
first and then tried to *call it as a constructor*, which failed on every attempt
— so the system silently fell back to slow REST polling.

**Fix:** Select the websocket class explicitly and require that the candidate is
an actual class (`isinstance(cand, type)`). The real-time fill stream came up
immediately and held a stable connection.

---

## 6. DH-905 — every order rejected for a bad exchange segment

**Symptom:** With the WebSocket finally live, **every** mirror order was rejected
by the broker with `DH-905: Input_Exception — bad values for parameters`. The
convergence logs showed the correct entry *and* exit being computed; the orders
just died at placement.

**Root cause:** The order-update WebSocket delivers the venue as **two separate
fields** (`exchange` + `segment`, e.g. `NSE` + `D`), whereas the REST path sends a
single ready `exchangeSegment`. The parser only read the REST field, so it
forwarded an **unknown segment** and the broker rejected everything.

**Fix:** Map `(exchange, segment)` → the correct composite segment
(e.g. derivatives on the primary exchange → the F&O segment). Entries and exits
began placing on the fast path in milliseconds.

> This was *the* blocker behind "entry works, exit never" — it rejected both
> equally. Entries only *appeared* to work because the slow reconciler placed them
> using position data that already carried a valid segment.

---

## 7. Wrong product type on corrections

**Symptom:** Even after the segment fix, a stuck short wouldn't close, and the
reconciler's breaker kept tripping.

**Root cause:** The retailer trades in **MARGIN (carry-forward)** product, but the
reconciler placed every correction as **INTRADAY**. An intraday buy cannot net off
a margin short — it opens a *fresh* intraday long instead, so the position never
closes and the breaker trips.

**Fix:** Corrections are now placed in the **position's own product** (the
follower's product for a close, the master's for a fresh entry), and the product
is carried through the pushed snapshot. A stuck leg was verified closing live, and
the operator returned flat.

---

## 8. Hot-path REST calls (the latency tax)

**Symptom:** Even when it worked, each mirror felt slow.

**Root cause:** On *every* event the follower made **two blocking REST calls** —
one to read its own net position before convergence, one for a margin check —
before it could place the order.

**Fixes:**
- **In-memory net cache.** The follower keeps its net position in memory,
  refreshed off the hot path and updated the instant an order is placed. The
  convergence read is now microseconds instead of 100–2500 ms.
- **Margin bypass on exits.** A close frees margin and must never be blocked, so
  exits skip the margin check entirely; entries read a cached balance.

Hot-path cost dropped to effectively just the broker's `place_order` round-trip.

---

## 9. Observability — the Trade Logger & Telegram alerts

To make all of the above *visible* rather than inferred from scattered logs:

- A **Trade Logger** tab on the dashboard shows, per event, what the retailer did
  and what the operator did, with the latency of every hop — so a slow entry or a
  non-`PLACED` exit is immediately obvious.
- **Telegram alerts** announce every entry/exit to a group with the operator's
  result and a "double-check the operator exited" nudge — a human-in-the-loop
  backstop on top of the automated layers.

---

## Testing approach

Pure decision logic (convergence math, drift detection, confirm-twice, circuit
breaker, hedge/price filtering, segment mapping, entry/exit labelling) is covered
by **offline unit tests** that stub the broker SDK — so the money-moving logic can
be verified without touching a live account. Every fix in this log shipped with
its own passing test before deployment behind the kill switch.
