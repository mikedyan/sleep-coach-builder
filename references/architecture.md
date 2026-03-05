# Architecture

Deep dive into the sleep coach system design. Read this when building the state management and CLI.

## Table of Contents

1. [System Overview](#system-overview)
2. [The State Machine](#the-state-machine)
3. [Gap-Based Escalation Model](#gap-based-escalation-model)
4. [State Schema](#state-schema)
5. [CLI Commands](#cli-commands)
6. [The Dispatcher Pattern](#the-dispatcher-pattern)
7. [Priority Order](#priority-order)
8. [Session Lifecycle](#session-lifecycle)
9. [Nightly Reset](#nightly-reset)

---

## System Overview

```
┌─────────────────────────────────────────────────────┐
│                   CRON LAYER                        │
│                                                     │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Dispatcher   │  │ Morning  │  │  Enforcement │  │
│  │ */2 active hr │  │ Report   │  │  */2 active  │  │
│  │ (cheap model) │  │ (daily)  │  │ (cheap model)│  │
│  └──────┬───────┘  └────┬─────┘  └──────┬───────┘  │
│         │               │               │           │
└─────────┼───────────────┼───────────────┼───────────┘
          │               │               │
          ▼               ▼               │
┌─────────────────────────────────────┐   │
│     TOPIC SESSION (smart model)     │   │
│   Dedicated chat topic/thread       │   │
│                                     │   │
│  • Receives context via session msg │   │
│  • Sends messages to user           │   │
│  • Updates state via CLI            │   │
│  • Handles conversation naturally   │   │
│  • Registers user responses (!)     │   │
└──────────────┬──────────────────────┘   │
               │                          │
               ▼                          ▼
┌─────────────────────────────────────────────────────┐
│                  STATE LAYER                        │
│                                                     │
│  state.json     — single source of truth            │
│  sleep_coach.py — CLI for all state mutations       │
│  lib/state.py   — locking, validation, derivation   │
│  enforce.py     — phone calls (reads + writes state)│
└─────────────────────────────────────────────────────┘
```

**Three actors** mutate state:
1. The topic session agent (via CLI) — most state changes
2. The dispatcher cron (via CLI) — only `context` reads
3. The enforcement script — L3/L4 call records

File locking ensures they don't corrupt each other.

---

## The State Machine

Four states representing the nightly progression:

```
INITIAL ──(user responds)──→ ENGAGED ──(plan_json)──→ HAS_PLAN ──(in_bed)──→ IN_BED
                                ↑                         │
                                └──(invalidate_plan)──────┘
```

### INITIAL
Night starts. No contact yet. The system sends the first outreach message.

### ENGAGED
The user has responded at least once. The agent pushes for a bedtime plan — "What's your plan for tonight? When are you aiming for bed?"

### HAS_PLAN
The user committed to a plan with an expected bedtime and optional transition checkpoints. The system monitors:
- Transition checkpoints (e.g., "Movie ends at 10:45" → check in at 10:45)
- Bedtime check (3 min before expected bedtime)
- Escalation if user goes silent

The plan can be invalidated (user changes plans, falls behind schedule) → back to ENGAGED.

### IN_BED
User confirmed they're in bed. Session complete. All crons return null, no more messages.

---

## Gap-Based Escalation Model

The core innovation. **Derive escalation level from one value: how long since the user last responded.** No counters, no flags, no stored level.

### Why Not Counters?

Earlier versions tracked `messages_sent_today`, `responses_received_today`, and an `awaiting_response` flag. This failed because:
- The flag could get out of sync (the #1 failure — see lessons-learned.md)
- Counters drifted when multiple components updated them
- Edge cases around midnight resets corrupted state

### How It Works

**Two fields in state:**
```json
{
  "last_response_at": "2026-02-10T21:30:00-08:00",
  "last_acted_level": 0
}
```

**Every 2 minutes, the cron computes:**
```
if last_response_at is null → level = 0 (guard: no escalation before first contact)

gap = now - last_response_at
level = level_from_gap(gap, thresholds)

if level > last_acted_level → take action, update last_acted_level
else → do nothing
```

**When user responds:**
```
last_response_at = now
last_acted_level = -1 or 0 (see below)
```

### last_acted_level Semantics

- `-1` = User just responded AND there's something proactive to say (continue conversation, push for plan, check transition). Next cron cycle will fire L0 action.
- `0` = Message was sent or nothing to say. Waiting for response or next threshold.
- `1-4` = Escalation level that was acted on.

When the user responds:
- If in INITIAL/ENGAGED state → set to -1 (agent has something to say)
- If in HAS_PLAN with a due transition → set to -1 (transition check needed)
- If in HAS_PLAN with nothing due → set to 0 (silent monitoring)

### Threshold Tables

Use two sets of thresholds based on when silence started (not current time):

**Relaxed (before threshold switch hour, e.g., 10 PM):**
```
L1 = 10 min    Gentle nudge
L2 = 20 min    Concerned check-in
L3 = 30 min    Phone call (if enabled)
L4 = 40 min    Partner contact (if enabled)
```

**Urgent (after threshold switch hour):**
```
L1 = 5 min     Gentle nudge
L2 = 10 min    Concerned check-in
L3 = 15 min    Phone call (if enabled)
L4 = 20 min    Partner contact (if enabled)
```

**Important:** Use the time silence STARTED (last_response_at) to pick the threshold table, not the current time. This prevents threshold jumps at the boundary — if the user stopped responding at 9:50 PM, use relaxed thresholds for the entire gap even after 10 PM.

### Key Properties

- **Stateless derivation** — Level is always computed, never stored. No drift.
- **Instant de-escalation** — Any response resets everything immediately.
- **Idempotent** — Multiple cron runs at the same level do nothing.
- **Monotonic** — Levels only go up within a silence period.

---

## State Schema

```json
{
  "schema_version": 3,
  "date": "2026-02-10",

  "state": "INITIAL",

  "plan": {
    "summary": null,
    "steps": [],
    "transitions": [
      {
        "check_at": "2026-02-10T22:45:00-08:00",
        "context": "Movie done — starting wind-down?",
        "checked": false,
        "result": null,
        "checked_at": null
      }
    ],
    "expected_bedtime": "2026-02-10T23:00:00-08:00",
    "bedtime_check_at": "2026-02-10T22:57:00-08:00",
    "created_at": null
  },

  "escalation": {
    "last_response_at": null,
    "last_acted_level": 0,
    "l3_call": {
      "attempted_at": null,
      "call_sid": null,
      "status": null,
      "duration_seconds": null
    },
    "l4_call": {
      "attempted_at": null,
      "call_sid": null,
      "status": null,
      "duration_seconds": null
    }
  },

  "in_bed": {
    "claimed": false,
    "claimed_at": null,
    "claim_text": null,
    "trusted": false,
    "trust_reason": null
  },

  "session": {
    "started_at": null,
    "ended_at": null,
    "end_reason": null,
    "success": null
  },

  "config": {
    "active_start": "21:15",
    "target_bedtime": "23:00",
    "hard_deadline": "04:00",
    "reset_hour": 4,
    "escalation_thresholds_relaxed": [10, 20, 30, 40],
    "escalation_thresholds_urgent": [5, 10, 15, 20],
    "threshold_switch_hour": 22,
    "trust_threshold_min": 10,
    "grace_period_seconds": 90
  },

  "last_updated": null,
  "last_updated_by": "system"
}
```

### Field Notes

- **`last_response_at = null`** — Special guard state. When null, derived level is always 0. Prevents escalation before first contact.
- **`transitions`** — Agent-generated checkpoints with ISO timestamps. Python stores them verbatim. The agent computes timestamps from natural language ("watching a movie for an hour" → now + 60 min).
- **`bedtime_check_at`** — Auto-calculated as expected_bedtime minus 3 minutes.
- **`l3_call` / `l4_call`** — `attempted_at` acts as dedup key. Non-null = already called tonight.
- **`grace_period_seconds`** — If state was updated very recently (< 90s), the enforcement cron skips. Prevents race conditions when the agent is mid-conversation.

---

## CLI Commands

The CLI is the single interface for all state mutations.

| Command | Effect |
|---|---|
| `context` | Read-only. Returns JSON with state, derived level, action_needed. |
| `response` | User responded → reset escalation, advance state machine. |
| `outreach_sent` | First message sent → set last_response_at, start escalation clock. |
| `escalation_acted <N>` | Mark escalation level N as acted on. |
| `plan_json <json>` | Set bedtime plan → transition to HAS_PLAN. |
| `in_bed [text]` | Mark user in bed → transition to IN_BED, end session. |
| `transition_checked [idx]` | Mark a plan transition as checked. |
| `bedtime_checked` | Clear the bedtime check trigger. |
| `invalidate_plan [reason]` | Back to ENGAGED state (plans changed). |
| `status` | Human-readable status dump. |

### The `context` Command Output

```json
{
  "current_time": "22:30",
  "state": "HAS_PLAN",
  "escalation_gap_minutes": 12.3,
  "derived_escalation_level": 0,
  "last_acted_level": 0,
  "last_response_at": "2026-02-10T22:17:42-08:00",
  "in_bed": false,
  "plan_summary": "Movie then bed",
  "expected_bedtime": "2026-02-10T23:00:00-08:00",
  "pending_transitions": 1,
  "action_needed": null,
  "action_reason": "Has plan, no transitions due — monitoring silently"
}
```

`action_needed` values: `first_outreach`, `check_in`, `push_for_plan`, `transition_check`, `bedtime_check`, `escalation_L1` through `escalation_L4`, or `null`.

---

## The Dispatcher Pattern

The cron layer uses a **dispatcher pattern** to bridge isolated cron agents with the persistent topic session.

### Why Not Direct Messaging?

Cron agents are ephemeral — they die after executing. If a cron sends a message directly to the user, the user's reply lands in the topic session which has zero context about what was asked.

### How It Works

1. Dispatcher cron runs `sleep_coach.py context`
2. If action is needed, it uses `sessions_send` to inject context into the topic session
3. The topic session receives the context, sends the appropriate message to the user
4. When the user replies, it naturally arrives in the topic session (which has the full conversation history)
5. The topic session processes the reply and updates state

This keeps the "smart" work in the topic session and the "dumb" checking in the cron.

### First Message Bridge

The topic session's first message in a session (triggered by `sessions_send` from the cron) needs to be sent via the `message` tool explicitly. After the user replies to that message, subsequent messages are delivered naturally through the channel.

---

## Priority Order

When the `context` command determines what action is needed, it follows this priority:

```
1. IN_BED → null (session complete, nothing to do)
2. OUTSIDE ACTIVE HOURS → null
3. FIRST OUTREACH (last_response_at is null) → first_outreach
4. ESCALATION (derived_level > last_acted_level):
   - If L0: state-dependent proactive action (continue convo, push plan, transition check)
   - If L1-L4: escalation action
5. HAS_PLAN SUPPRESSION: If plan exists and no transitions are due → null (silent monitoring)
6. WAITING: If last_acted >= derived_level → null (already acted, waiting)
```

**HAS_PLAN suppression is critical.** When the user has made a plan and no checkpoints are due, the system goes silent. Having a plan IS the engagement — don't nag someone who already committed to a bedtime.

---

## Session Lifecycle

A typical successful night:

```
21:15  State resets. INITIAL.
       Dispatcher → context → first_outreach → sends to topic session.
       Topic session greets user.

21:17  User replies "hey, just got home"
       Topic session calls `response` → ENGAGED. last_acted = -1.

21:18  Dispatcher → context → push_for_plan (L0, last_acted was -1).
       Topic session: "What's the plan tonight? When are you aiming for bed?"

21:20  User: "Watching a show, should be done by 10:30, bed by 11"
       Topic session calls `response`, then `plan_json` with transitions.
       State → HAS_PLAN.

22:30  Transition due. Dispatcher → context → transition_check.
       Topic session: "Show should be wrapping up — starting the wind-down?"

22:31  User: "Yeah, brushing teeth now"
       Topic session calls `response`, marks transition checked.

22:57  Bedtime check due. Dispatcher → context → bedtime_check.
       Topic session: "Almost 11 — heading to bed?"

22:58  User: "In bed, goodnight"
       Topic session calls `in_bed`. State → IN_BED.
       All cron checks return null. Silence until morning.
```

A silence scenario:

```
21:20  Outreach sent. User doesn't reply.
21:30  10 min gap → L1. Gentle nudge sent.
21:40  20 min gap → L2. Concerned check-in sent.
21:50  30 min gap → L3. enforce.py calls user's phone.
22:00  40 min gap → L4. enforce.py calls partner.

22:05  User replies: "sorry was in the shower"
       Topic session calls `response`.
       last_response_at = now, last_acted_level = -1.
       Everything resets. Normal conversation resumes.
```

---

## Nightly Reset

The state file resets daily to start fresh:

- **Reset trigger:** `load_state()` checks if `state.date != today AND current_hour >= reset_hour`
- **Reset hour:** Default 4 AM. Configurable. Must be after the hard deadline.
- **Between midnight and reset hour:** Yesterday's state persists. Session continuity across midnight.
- **Archive:** Before resetting, the old state is saved to `state-archive/YYYY-MM-DD.json`
- **Fresh state:** All fields reset, state = INITIAL, last_response_at = null

The morning report cron runs BEFORE the reset hour (e.g., 8 AM, reset at 9:15 PM next day), so it always has access to last night's plan in the live state file.
