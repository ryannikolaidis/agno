# Test Log: interfaces/agui

> Per-file test results below (PASS / PENDING).

### agent_with_media.py

**Status:** PASS

**Description:** Agent With Media - AG-UI agent (Google Gemini) that accepts multimodal user input.

**Result:** AgentOS boots with the AG-UI interface; /config returns 200. Media sent through POST /agui reaches the Gemini agent, which describes it accurately - verified with an image via CLI and interactively in a browser. Multimodal input path verified end-to-end.

---

### agent_with_silent_tools.py

**Status:** PENDING

**Description:** Silent External Tools - Suppress verbose messages in frontends.

---

### agent_with_tools.py

**Status:** PENDING

**Description:** Agent With Tools.

---

### basic.py

**Status:** PENDING

**Description:** Basic.

---

### multiple_instances.py

**Status:** PENDING

**Description:** Multiple Instances.

---

### reasoning_agent.py

**Status:** PENDING

**Description:** Reasoning Agent.

---

### research_team.py

**Status:** PENDING

**Description:** Research Team.

---

### state_events.py

**Status:** PENDING

**Description:** Outbound state synchronization via STATE_SNAPSHOT + STATE_DELTA events. Emits initial and final STATE_SNAPSHOT events plus STATE_DELTA JSON Patch ops after each state-mutating tool call.

---

### structured_output.py

**Status:** PENDING

**Description:** Structured Output.

---

### workflow_progress.py

**Status:** PASS

**Description:** Native-first workflow progress over AG-UI -- a sequential Workflow (research -> analyze -> summarize) whose live progress renders from `state.workflow_progress.steps` ({id, name, status, output}) via STATE_SNAPSHOT/STATE_DELTA + native STEP_STARTED/FINISHED, with ZERO structural CustomEvent. The "simple case" unlocked by the native-first rework.

**Result:** Verified end-to-end. The sequential workflow's progress renders live from `state.workflow_progress.steps` in the AG-UI Dojo (agentic_generative_ui feature, via useCoAgentStateRender) -- the panel fills research -> analyze -> summarize to 3/3 Complete -- with no custom-event handling on the client. Raw SSE confirms the wire: STATE carries workflow_progress.steps, native STEP fires, zero structural CustomEvent.

**Verification:**
- Unit: 98 agui interface tests pass (workflow + router + state_events); 5/5 key mutations re-proven (custom_event exclusion, _finalize_run re-inject, mark_completed promotion, strip on BOTH save paths, enum-driven RAW coverage); cheat-detector clean; ruff clean; mypy 0 introduced (base-comparison vs #8364). (DONE)
- Core: 410 workflow tests pass (7 pre-existing skips) -- transient strip non-regressing on save/load. (DONE)
- Raw SSE (4.1): POST /agui -> 2 STATE_SNAPSHOT, 7 STATE_DELTA, 3 STEP_STARTED, 3 STEP_FINISHED, CUSTOM=0; workflow_progress.steps on the wire. (DONE -- PASS)
- Live render (4.2): state.workflow_progress.steps renders and updates in the Dojo; 3/3 Complete observed (screenshot on file). (DONE -- PASS)
- Robustness (4.3): loop/parallel/condition/router/nested populate the flat steps[] with no structural CUSTOM/RAW; step_error -> "error" (no RUN_ERROR); cancel -> "cancelled"; pause -> "paused"; no-state baseline emits a leading STATE_SNAPSHOT (real-engine unit tests). Concurrency isolation: two sessions run concurrently with no progress bleed. (DONE -- PASS)
- A/B (4.4): agent/team AG-UI paths unchanged -- 15 router + state_events tests pass. (DONE -- PASS)

**Known gaps (honest):**
- Postgres not run live -- the transient strip is backend-agnostic by construction (pops the key before the DB driver in both save_session/asave_session); verified on sqlite sync + async, not Postgres.
- Topology grouping (parallel/loop/condition/router rendered flat, not nested) deferred to the follow-up.
- Interactive pause/resume (HITL) deferred; pause shows as a status label only.

---

### activity_events.py (2026-07-09)

**Status:** PASS (wire contract)

**Description:** Workflow progress dual-emitted as opt-in AG-UI ACTIVITY events (`AGUI(emit_activity=True)`): ACTIVITY_SNAPSHOT first, RFC 6902 ACTIVITY_DELTA per step transition, full snapshot at every terminal (completed and error), stable per-run message id `agno-workflow-progress-<run_id>`, activity_type `agno-workflow-progress`. Flag off keeps the wire byte-identical.

**Result:** Wire contract verified in-process (ASGI TestClient, deterministic function-step workflows -- no live model). Completed run: `... [STEP_STARTED -> STATE_DELTA -> ACTIVITY_DELTA]xN -> STATE_SNAPSHOT -> ACTIVITY_SNAPSHOT(COMPLETED) -> RUN_FINISHED`. Hard error (`on_error="fail"`): `... STATE_SNAPSHOT(ERROR) -> ACTIVITY_SNAPSHOT -> RUN_ERROR`, nothing after. Flags-off error sequence byte-identical to the pre-change live capture.

**Verification:**
- Unit: 9 dedicated tests (test_agui_activity.py) incl. default-off absence regression, snapshot-first, delta replay vs terminal, STATE-before-ACTIVITY per tick, both error shapes, agent inertness, camelCase HTTP wire keys. Full agui interface suite green. (DONE -- PASS)
- Raw SSE (in-process): captures on file (happy/error x flags off/on); all ordering assertions pass. (DONE -- PASS)
- Dojo live (2026-07-10, deterministic function-step rig with the flag ON): Task Progress card renders unchanged (STATE channel untouched) and the deployed CopilotKit client parses the full flag-on stream -- the run completes with ACTIVITY_SNAPSHOT/ACTIVITY_DELTA events visible on the browser wire (devtools Network) and no stream kill. As designed, nothing renders for the activity itself: no page registers a renderer for `agno-workflow-progress`. (DONE -- PASS)

**Known gaps (honest):**
- ACTIVITY is wire-only until a client registers a renderer; the registration snippet ships in the docstring, and the dojo renderer itself is a separate ag-ui PR.
- Live gpt-5.5 serve of this exact cookbook file not run (the dojo cell used an equivalent deterministic workflow through the same AGUI flag path).

---

### session_rehydration.py (2026-07-09)

**Status:** PASS (wire contract)

**Description:** Session-history rehydration via opt-in MESSAGES_SNAPSHOT (`AGUI(emit_messages_snapshot=True)`): one snapshot at run start (after RUN_STARTED + initial STATE_SNAPSHOT, before all streamed traffic) replaying the session's prior turns, gated four ways (flag, fresh-run-only, non-empty server history, no assistant message in the payload) with the just-typed user message echoed at the snapshot tail under a re-minted id.

**Result:** Wire contract verified in-process (ASGI TestClient, DB-backed deterministic workflow). Run 1 (fresh thread): no snapshot. Run 2 (same thread): exactly one MESSAGES_SNAPSHOT at position `RUN_STARTED -> STATE_SNAPSHOT -> MESSAGES_SNAPSHOT -> ...`, replaying turn 1 as user/assistant and echoing the new user message last with a fresh id. Session-read failure logs and continues (no RUN_ERROR).

**Verification:**
- Unit: 7 dedicated tests (test_agui_messages_snapshot.py) incl. each gate individually, agent-path mapping via a stub model, workflow interaction synthesis, and the swallow-on-DB-failure guarantee. Full agui interface suite green. (DONE -- PASS)
- Raw SSE (in-process): two-run capture on file; position + tail-echo assertions pass. (DONE -- PASS)

- Live server (2026-07-10): same-thread two-run curl against a running rig reproduces the exact contract -- run 1 no snapshot; run 2 one MESSAGES_SNAPSHOT at `RUN_STARTED -> STATE_SNAPSHOT -> MESSAGES_SNAPSHOT -> ...` replaying turn 1 with the re-minted tail echo. (DONE -- PASS)
- Gate observed live in the dojo (2026-07-10): the stock dojo mints a FRESH threadId on every page reload (never reattaches), so the backend correctly emits no snapshot there -- the fresh-thread/non-empty-history gates working as designed.

**Known gaps (honest):**
- The visual CopilotKit merge of a snapshot could not be exercised in the stock dojo (no thread persistence across reloads -- a client-side property). Risk is bounded by construction: the gates ensure only history-less clients ever receive a snapshot, so there is nothing on screen to reorder; at worst the optimistic user bubble remounts once with identical content.
- Text-only v1: no tool-call replay, no reasoning messages, no media in the snapshot (documented in the module docstring).

---
