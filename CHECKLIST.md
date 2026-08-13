# Assessment Checklist — Green Lot Cupping & Approval

Mapped against every item in the take-home brief.

## Deliverables the brief asks for

| # | Requirement | Status | Notes |
|---|-------------|--------|-------|
| 1 | **Design the data model yourself** (lots + cupping sessions) | ✅ Done | Two forms: **Coffee Lot** and **Cupping Session** (one lot → many sessions). SCA total auto-computed. |
| 2 | **Two-level approval workflow** (quality, then commercial) | ✅ Done | **QC Review** → **Commercial Sign-off**, driven by status + decision fields on the lot. *Trade-off:* built form/status-driven, not the BPMN engine (see log, challenge #3). |
| 3 | **A dashboard** | ✅ Done | Home page: hero + workflow guide + **two live native charts** (donut: BUY/HOLD/REJECT; bar: avg score by lot). Data-driven via SQL → auto-updates. |
| 4 | **A genuinely agentic AI agent** (multi tool calls, reasoning, decision branch, write-back) | ✅ Built & verified | **Lot Screening Agent**: 3 autonomous SQL tool calls (read sheets → read comparable lots → UPDATE lot), computes avg/spread, branches BUY/HOLD/REJECT, writes decision + rationale back to the DB. All 3 branches verified. ⚠️ *Live re-run currently blocked by a Joget platform bug* (see below); results are persisted in the app. |
| 5 | **Use OpenRouter free credits** for the LLM | ✅ Done | Model: `nvidia/nemotron-3-super-120b-a12b:free` via OpenRouter (free tier). |
| 6 | **Extend the front-end / make it yours** | ✅ Done | Custom branded hero with personalized greeting, quick-nav, color-coded workflow cards, native charts. |
| 7 | **Export app as `.jwa`** | ✅ Done | Exported with form data bundled. ⚠️ Re-export after the latest UI polish so the file is current. |
| 8 | **Short screen recording — AI agent running** | ⏳ Your task | Blocked right now by the platform bug (below). Options: retry once the plugin is patched, or record the persisted results as the fallback (see README). |
| 9 | **Short written log** (tried / worked / didn't / routes dropped) | ✅ Done | `BUILD_LOG.md`. |
| 10 | **This README + checklist** | ✅ Done | `README.md` + this file. |

## The one open blocker 

- **Joget platform bug** in `agent-builder-plugin-8.2.2`: its `AgentTaskMetadata.useMemory` field can't be read by Joget's JSON deserializer, so **any** agent run (even a brand-new trivial one) errors before it starts. Proven not to be our config. Fix is Joget's — update the plugin via **System Settings → Manage Plugins**, or wait for the hosted instance to patch. Full detail in `BUILD_LOG.md` challenge #6.

## Nice-to-haves considered / dropped
- Full BPMN workflow with an inbox + auto approve/reject buttons — dropped ; form/status pattern used instead.
- In-app "Run AI Screening" button — not added because the agent runtime is blocked; would be wired via a Run-Process menu once the platform bug clears.