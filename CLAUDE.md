# CLAUDE.md — GTM Lead-Gen & ICP Scoring Automation

Operating guide for any Claude/agent working in this repo. Read this **before** touching the
workflow. The most important section is [§8 LESSONS](#8-lessons--learn-from-mistakes): it is a
living log you must read at the start of a task and append to whenever something breaks.

---

## 1. What this project is

An n8n workflow that finds, enriches, scores, and routes African SMEs likely to buy WhatsApp/voice
AI agents. See `ARCHITECTURE.md` for the full design. Markets: Nigeria (primary), Kenya, South
Africa, Ghana, Egypt, Rwanda. Owner contact / digest recipient: `smarworld25@gmail.com`.

## 2. Golden rules (do not violate)

1. **Never hardcode secrets.** Credentials by name only. Tunables live in the Config node.
2. **Dedup is a DB guarantee, not a vibe.** The Supabase unique index is the source of truth.
   Do not remove it "temporarily."
3. **A branch is not done until it has produced a real row end-to-end.** Config that looks correct
   proves nothing. Run it. Watch the row land in Supabase and Airtable.
4. **Claude scoring output is JSON only.** No prose, no markdown fences. Always parse in try/catch.
   A parse failure routes the item to `rejected`, it never crashes the batch.
5. **Fail loud to `run_log`, continue soft in the flow.** continue-on-fail + retry on every
   external call. One bad item must not kill the daily run.
6. **Idempotent always.** Re-running today's batch must not create duplicates anywhere.

## 3. Build / run workflow

- Import workflow JSON into n8n Cloud. Set all credentials by name (see `ARCHITECTURE.md §10`).
- Apply Supabase DDL for `leads`, `rejected`, `run_log` (including the unique dedup index).
- Create the Airtable base + fields per `ARCHITECTURE.md §7`.
- First run: use the **Manual** trigger with `PER_SOURCE_CAP = 5` to smoke-test cheaply before
  enabling the daily Schedule.

## 4. Verification checklist (run after ANY change)

- [ ] Manual run with small caps completes without red nodes.
- [ ] At least one lead reaches Supabase `leads` **and** the matching Airtable row.
- [ ] Re-run the same batch → **zero** new rows (idempotency holds).
- [ ] Force a bad Claude response (malformed JSON) → item lands in `rejected`, batch survives.
- [ ] `run_log` captures a deliberately failed external call.
- [ ] Digest email arrives with correct counts per source/country.

## 5. Claude scoring node contract

- Model `claude-sonnet-4-6`, `max_tokens: 1000`.
- System prompt must end with: *"Respond with ONLY a valid JSON object. No preamble, no markdown,
  no code fences."*
- Output must match the schema in `ARCHITECTURE.md §6`. If you change the rubric weights, update
  BOTH the system prompt and `ARCHITECTURE.md §6` — they must never drift.

## 6. Data hygiene

- Country codes are 2-letter ISO: `NG KE ZA GH EG RW`. Phone validation checks the matching
  dialing code (+234 +254 +27 +233 +20 +250).
- Segment tags are a closed set (`ARCHITECTURE.md §4`). Do not invent new tags in the scoring
  node without adding them to the enum in both places.

## 7. Common tasks

- **Add a source:** new branch → normalize to common schema → into Merge. Nothing downstream
  changes. Verify per §4.
- **Tune the threshold:** change `SCORE_THRESHOLD` in Config only.
- **Tune the rubric:** edit weights in the scoring system prompt AND `ARCHITECTURE.md §6`.

---

## 8. LESSONS — learn from mistakes

> **Protocol.** At the START of any task, read this whole section. When anything breaks — a bug,
> a wrong assumption, a wasted hour — STOP and append an entry using the template below before
> moving on. Never delete entries; supersede them. This is how the project stops repeating
> mistakes. Keep entries short, specific, and actionable.

**Entry template:**

```
### L-NNN — <one-line title>   (YYYY-MM-DD)
- Symptom: what went wrong / what I observed.
- Root cause: the real reason (not the surface error).
- Fix: what actually resolved it.
- Rule going forward: the durable takeaway (add to Golden Rules if load-bearing).
```

---

### L-001 — Trusting config instead of behavior   (seed)
- Symptom: a branch looked fully wired but produced zero rows in production.
- Root cause: the branch had never actually executed; config inspection hid a dead path.
- Fix: ran the branch with small caps and confirmed a row landed end-to-end.
- Rule going forward: a branch is not "done" until it has produced a real row. Verify by running,
  never by reading node config. (Now Golden Rule #3.)

### L-002 — Claude returned markdown fences and broke JSON.parse   (seed)
- Symptom: scoring node threw on ```json fences wrapping the output.
- Root cause: the system prompt didn't forbid fences hard enough; parser assumed clean JSON.
- Fix: strengthened the prompt ("ONLY valid JSON, no fences"), added fence-stripping + try/catch,
  routed parse failures to `rejected` with reason `score_parse_error`.
- Rule going forward: never assume clean JSON from a model. Strip fences, parse defensively, fail
  the item not the batch. (Now Golden Rule #4.)

### L-003 — Duplicate leads across re-runs   (seed)
- Symptom: re-running the daily batch created duplicate rows in Supabase and Airtable.
- Root cause: dedup was enforced only in-flow; concurrent/re-runs slipped past it.
- Fix: added a Supabase unique index on the normalized key; Airtable upsert keyed on
  `supabase_lead_id`.
- Rule going forward: dedup must be a DB-level guarantee, not just an in-flow filter.
  (Now Golden Rule #2 / #6.)

### L-004 — One bad Apify item killed the whole run   (seed)
- Symptom: a single malformed scrape result aborted the batch; no leads processed that day.
- Root cause: external-call nodes fail-fast by default; no error isolation.
- Fix: enabled retry + continue-on-fail on all external nodes; every failure writes to `run_log`;
  digest surfaces the error count.
- Rule going forward: fail loud to the log, continue soft in the flow. (Now Golden Rule #5.)

### L-005 — Wrong owner email hardcoded   (seed)
- Symptom: risk of digests sent to a nonexistent address.
- Root cause: address typed inline in the Gmail node from memory.
- Fix: digest recipient lives in Config (`DIGEST_RECIPIENT = smarworld25@gmail.com`). The address
  `olayeleawujoola@gmail.com` does NOT exist — never use it.
- Rule going forward: no addresses/secrets inline; pull from Config. Correct email is
  `smarworld25@gmail.com`.

<!-- Append new lessons below this line -->
