# Green Lot Cupping & Approval — Build Log

A Joget DX 9 app that replaces the spreadsheet/email process a specialty roastery uses to
score green coffee lots (SCA cupping) and decide whether to buy them.

---

## 1. The app in one paragraph

Green (unroasted) coffee **lots** are sourced from producers/cooperatives. Each lot is
**cupped** (SCA-scored, 0–100) by one or more cuppers before the roastery commits to buying.
The app models that: buyers enter lots, cuppers file SCA score sheets, an **AI agent
pre-screens** each lot and recommends **BUY / HOLD / REJECT**, then a **two-level human
approval** (Quality Control, then Commercial) makes the call.

**Data model**
- `Coffee Lot` — producer, origin, variety, process, altitude, green kg, price/kg, plus
  system-maintained screening fields (status, avg score, cupper spread, session count, AI
  decision, AI rationale) and an Approvals section (QC decision + comments, Commercial
  decision + comments).
- `Cupping Session` — one cupper's SCA sheet: the 10 attributes (fragrance/aroma, flavor,
  aftertaste, acidity, body, balance, uniformity, clean cup, sweetness, overall), a defect
  deduction, and an **auto-computed total** (Calculation Field), linked to a lot.

**Demo data shipped in the .jwa:** 3 lots (Ethiopia, Colombia, Brazil) and 7 cupping sheets,
deliberately built to exercise all three AI decisions.

---

## 2. The AI agent (the part I spent the most thought on)

Built with Joget's **AI Agent Builder** on **OpenRouter**. It is genuinely *agentic*, not a
one-shot prompt:

Given a lot id it, on its own:
1. **Calls a SQL tool** to pull every cupping sheet for that lot.
2. **Calls the tool again** to pull the other lots for price/score benchmarking.
3. **Reasons** over the data — computes the average total, the cupper *spread* (max − min),
   the session count, and scans notes for flagged defects.
4. Applies a **decision rubric** (REJECT if avg < 80; HOLD if avg ≥ 80 but spread > 4 or a
   defect was flagged; BUY if avg ≥ 80, spread ≤ 4, no defect) → picks **BUY / HOLD / REJECT**.
5. **Calls the tool a third time to UPDATE the lot record** — writing the decision, a
   quantitative rationale, the average, spread, session count, and status back to the DB.

So: multiple autonomous tool calls, reasoning over data it fetched itself, a real decision
branch, and a write-back. Verified end-to-end for all three branches:

| Lot | Avg | Spread | Agent decision | Why (agent's own words, abridged) |
|-----|-----|--------|----------------|-----------------------------------|
| ETH-2026-014 | 86.83 | 1.5 | **BUY** | tight agreement, good value vs comparable lots |
| COL-2026-007 | 83.75 | 8.5 | **HOLD** | "defect noted and high variability → re-cup before purchase" |
| BRA-2026-031 | 74.50 | 3.0 | **REJECT** | "below 80 … defect notes indicate past-crop character" |

The HOLD case is the one I'm happiest with: the agent caught both the 8.5-point cupper
disagreement *and* the fermented-taint note one cupper left, and recommended re-cupping —
exactly the judgment a head of QC would want a screen to make.

---

## 3. What worked well

- **Forms via JSON.** The form builder's JSON-definition editor round-trips cleanly, so I
  defined both forms (incl. the SCA Calculation Field and options binders) as JSON and
  applied them in one shot instead of dragging dozens of fields.
- **The Calculation Field** auto-summing the 10 SCA attributes minus defects — instant,
  correct totals persisted to the DB.
- **OpenRouter is a first-class credential provider** in AI Central Config, and the AI Agent
  Builder's generic **SQL tool** turned out to be a great fit: the LLM writes its own SELECT
  and UPDATE statements, which is what makes the flow genuinely agentic.

## 4. What didn't work first time — and the fixes

1. **OpenRouter retired the free tier of the good tool-calling models.** `deepseek-chat-v3`,
   `llama-3.3-70b`, `gemini-2.0-flash-exp` all failed ("unavailable for free" / "no
   endpoints"). Fix: queried OpenRouter's live `/models` API and filtered for `:free` models
   that actually support `tools`. `openai/gpt-oss-20b:free` connected and *called* the tool,
   but a 20B model didn't reliably chain read → compute → write. Switching to
   **`nvidia/nemotron-3-super-120b-a12b:free`** (also free) made it reliable — model size
   mattered more than prompt tuning here.
2. **Datalist definitions do NOT persist through the JSON editor** (forms do). Symptom: lists
   saved as empty. Fix: build lists through the visual builder (binder + drag columns), which
   persists correctly. Worth knowing — it's an easy trap.
3. **Two-level approval.** I started in the **BPMN Process Builder** (Start → AI Agent tool →
   QC → Commercial → End) — the AI Agent is even available as a process tool. Rather than ship
   a half-connected process, I pivoted to a **status/form-driven** two-level approval: the AI
   sets status = "AI Screened", QC and Commercial each have their own decision + comments
   section on the lot, reviewed via dedicated **QC Review** and **Commercial Sign-off**
   screens. Functionally two-level; see "routes I dropped" for the trade-off.
4. **Lazy DB columns.** Joget only adds a form field's column to the table on first save, so
   filtered queues referencing brand-new fields failed column introspection until I saved a
   lot once. Fix: saved a record to materialize the columns.
5. **DatePicker format quirk** rendered a broken placeholder and rejected ISO dates; I dropped
   the sample-date field rather than chase it (not core to the model).
6. **A mid-project platform update broke the live agent run.** After the agent was built and
   verified working (all three decisions written to the DB), the Joget cloud instance updated
   its AI Agent plugin. Live runs now fail immediately with:
   `Failed making field 'org.joget.ai.agent.model.AgentTaskMetadata#useMemory' accessible;
   either increase its visibility or write a custom TypeAdapter for its declaring type.`
   This is a **platform-side regression** — a Gson/Java reflection-access error deserializing
   Joget's *own* task-metadata model (`useMemory` is an internal field with no user setting).
   I proved it is not our configuration by building a **brand-new, minimal agent from scratch**
   (one OpenRouter LLM + one text prompt, nothing else) under the current plugin — it throws the
   **identical** `useMemory` error the moment its task has any content. Re-saving the agent and
   toggling every related option (incl. "Enable Context Window Management") also makes no
   difference, and the identical config ran cleanly earlier the same day. So the bug affects
   *every* agent on the instance, not our app. The probable fix is Joget's (patch/rollback the AI
   Agent plugin), and it may simply clear on its own when the hosted instance is next patched. **The agent's results survive it**: the BUY/HOLD/REJECT
   decisions, averages, spreads and rationales it computed are persisted in the Coffee Lot
   records (visible in the Coffee Lots list and each lot's "AI Screening Outcome" section) and
   are included in the exported `.jwa`.

## 5. Routes I considered and dropped (or deferred)

- **Full BPMN workflow engine + the Process Enhancement plugin's auto approve/reject buttons +
  a task inbox** — the "proper" Joget way and my first choice. Dropped only because the
  visual was not 
- **Filtered QC/Commercial queues** (via a datalist Extra Condition) — hit the lazy-column
  issue; used a shared list for both review screens instead.
- **AI Designer / generate-the-form-with-AI** — skipped deliberately; I wanted a hand-designed
  data model over an auto-generated one.
- **A dedicated charts dashboard** — **built** on the Home/Welcome page. The default Joget
  "Get Started" tutorial content was replaced (via the Page Components builder) with a real
  dashboard: an intro + "How it works" guide (HTML Code component), plus **two live,
  data-driven native charts** (Joget's Chart component, Apache ECharts): a **donut** of lots
  by AI recommendation (BUY/HOLD/REJECT) and a **bar chart** of average cupping score by lot.
  Both charts run off SQL queries against `app_fd_coffee_lot` (Default Datasource), so they
  **auto-update** whenever lot data changes — no manual refresh.
  *Config note:* the Chart component's "Using List" datasource wouldn't expose the datalist's
  columns as numeric measures (Number Value dropdown came up empty), so I switched each chart
  to **Default Datasource + a SQL query** (`GROUP BY c_ai_decision` for the donut; `CAST(
  c_avg_score AS DECIMAL)` for the bar) — full control and correct numeric aggregation.
  The Coffee Lots list remains the live tabular view of each lot's avg score + AI decision.

## 6. What's left for you

- 
---

## 7. How to record the AI agent running (2–3 min)

1. Design console → app **Green Lot Cupping & Approval** → **AI Agent Builder → Lot Screening
   Agent → Preview**.
2. In **Lot record id to screen**, paste a lot id and click **Run Agent**. Use an *un-screened*
   lot for a clean live run — e.g. re-run **BRA** to show a live **REJECT**:
   - ETH (BUY): `13291dc7-eaad-44d2-9b9a-35ca545d141f`
   - COL (HOLD): `7e4e9d50-b5d7-4fb6-9cce-4f1313aa3f98`
   - BRA (REJECT): `fdeb9ce9-c657-4307-a1c6-68611d221055`
3. Watch the **Execution Output**: it shows the agent entering the task and making the
   **Database Query** tool calls, then prints the final decision + rationale. (120B model,
   ~20–30s.)
4. Then open the running app → **Coffee Lots** and show that the **Average Cupping Score** and
   **AI Recommendation** columns were written by the agent — that's the write-back landing in
   the app.

*Note on the model:* it's a free OpenRouter model, so it occasionally composes a slightly
malformed 

### Fallback if the live run is still throwing the `useMemory` platform error

The live agent run currently fails due to the Joget platform regression described in challenge
#6 (out of our control — it may clear when Joget patches the instance; retry later). If you need
to record *now*, record the pipeline from the evidence that persists — this still fully
demonstrates the agentic design:

1. **Design view** — walk through the agent: OpenRouter LLM (nvidia/nemotron-3-super-120b),
   the SYSTEM persona + decision rubric, the USER task prompt (read → compute → decide → write),
   and the `run_sql` Database Query tool. Narrate that it makes multiple autonomous tool calls.
2. **Coffee Lots list** — show ETH = **BUY** (86.83), COL = **HOLD** (83.75), BRA = **REJECT**
   (74.50). These decisions were written by the agent, not typed by a human.
3. **Open the BRA lot → "AI Screening Outcome"** — show the agent's own **AI Rationale** text:
   *"Average score 74.50 is below 80, spread 3 over 2 sessions … defect notes indicate past-crop
   character, so REJECT."* That paragraph is the agent's output, persisted via its SQL UPDATE.
4. State plainly that a mid-project platform update broke the *live re-run* button, and that the
   persisted results are the proof of the completed run. (Honest, and it's a real engineering
   challenge — good material.)

---

*Draft — please review and edit before submitting; adjust the framing/authorship to taste.*
