# Lessons Learned

Hard-won lessons from 5 versions of development (v1 through v5, January–February 2026). Every major failure, its root cause, and the fix. **Read this before going live.**

## Table of Contents

1. [The Response Registration Bug](#1-the-response-registration-bug)
2. [Counter Drift](#2-counter-drift)
3. [False Escalation on Date Night](#3-false-escalation-on-date-night)
4. [The Context Gap](#4-the-context-gap)
5. [Token Overflow](#5-token-overflow)
6. [HAS_PLAN Nagging](#6-has_plan-nagging)
7. [Threshold Boundary Jumps](#7-threshold-boundary-jumps)
8. [Call Duration Threshold](#8-call-duration-threshold)
9. [Sleep Tracker Date Math](#9-sleep-tracker-date-math)
10. [Cron Spam](#10-cron-spam)
11. [Topic Session Has No Memory](#11-topic-session-has-no-memory)
12. [Grace Period Races](#12-grace-period-races)
13. [First Message Delivery](#13-first-message-delivery)
14. [Cron Agent Memory](#14-cron-agent-memory)

---

## 1. The Response Registration Bug

**Severity: Critical — caused every version from v1-v4 to malfunction**

### What Happened
The topic session agent failed to call `sleep_coach.py response` when the user sent messages. The system thought the user was ignoring it while they were actively chatting.

### Root Cause
The agent instructions didn't make it clear enough that EVERY user message needs a `response` call. The agent would have conversations, update plans, but never reset the escalation timer.

### Consequence
Escalation ran unchecked. The system called the user's phone while they were mid-conversation in the chat. In the worst case (v4), it called the user's partner at 12:28 AM on date night.

### Fix (v4)
Made response registration **non-negotiable** in agent instructions:
> "When [user] sends a message in this topic, you MUST call `sleep_coach.py response` immediately. If you fail to do this, the alarm will escalate to calling their partner. DO NOT FAIL."

### Fix (v5 — belt AND suspenders)
Added **state reconciliation** in the dispatcher cron: before dispatching, check `sessions_history` for user messages newer than `last_response_at`. If found, call `response` as a backup. Two paths to update = no single point of failure.

### Lesson
**The most important operation in the system is the simplest one.** Don't assume the agent will remember to do it. Make it impossible to miss:
- Use the strongest language in your agent instructions
- Add a backup mechanism that catches the failure
- Log when `response` is called so you can audit

---

## 2. Counter Drift

**Severity: High — made escalation unreliable in v1-v3**

### What Happened
v1-v3 tracked `messages_sent_today`, `responses_received_today`, and an `awaiting_response` flag. Multiple components updated these counters, and they drifted out of sync.

### Root Cause
The dispatcher cron, the topic session, and the enforcement script all touched the same counters. Race conditions and missed updates caused the counts to diverge from reality.

### Fix
Eliminated all counters. The gap-based model uses exactly TWO fields: `last_response_at` and `last_acted_level`. Escalation is derived from the time gap, not tracked. This is mathematically incapable of drifting.

### Lesson
**Derived state > stored state.** If you can compute something from a single timestamp, don't store it as a counter. Counters are shared mutable state — they WILL go wrong eventually.

---

## 3. False Escalation on Date Night

**Severity: Critical — the incident that got enforcement disabled**

### What Happened
On February 23, the system made a false L3 phone call to the user at 10 PM. The user answered (7 seconds), but the call duration was under the 10-second threshold, so the system considered it "unsuccessful" and cascaded to L4 — calling the user's partner at 12:28 AM.

### Root Cause (compound failure)
1. Response registration bug (again) — user was chatting but escalation timer wasn't reset
2. L3→L4 threshold too aggressive — 10-second call duration is too short for "unsuccessful"
3. No human-in-the-loop for L4 — the system called the partner automatically without confirmation

### Fix
1. Fixed response registration (v5 state reconciliation)
2. Increased call success threshold to 10+ seconds
3. **Enforcement crons disabled pending full review** — the right call. Better to have no enforcement than false enforcement.

### Lesson
**Phone calls to third parties are nuclear.** They should have the highest bar for triggering:
- Consider requiring confirmation before L4 (send a message: "Should I call [partner]?")
- Use generous call success thresholds (10+ seconds)
- Test extensively with dry-run mode before enabling
- Have a kill switch (ability to disable enforcement instantly)

---

## 4. The Context Gap

**Severity: Medium — architectural flaw in cron design**

### What Happened
Cron agents are isolated — they have no conversation history. When a cron sends a message directly to the user, the user's reply lands in the topic session which has zero context about what was asked.

### Root Cause
The cron dies after sending. It can't receive replies. The topic session receives the reply but doesn't know what prompted it.

### Fix
The **dispatcher pattern**: crons never message the user directly. Instead, they inject context into the topic session via `sessions_send`. The topic session then sends the message AND handles the reply.

### Lesson
**Crons are fire-and-forget.** Don't design systems where crons need to handle responses. Use crons for checking state and delegating, not for conversations.

---

## 5. Token Overflow

**Severity: Medium — caused cron failures**

### What Happened
Early cron prompts used `sessions_history(limit=50)` or even `limit=100` to get context. This blew past token limits for the cost-effective models running the crons.

### Fix
Cap at `sessions_history(limit=20)` for cron contexts. The dispatcher doesn't need deep history — it checks state via the CLI, not conversation context.

### Lesson
**Cron agents should be lean.** They run frequently (every 2 min), so token efficiency matters. Use the CLI for state, not conversation history.

---

## 6. HAS_PLAN Nagging

**Severity: Medium — annoying UX**

### What Happened
When the user had a plan with transitions not yet due, the system kept sending messages every 2 minutes because `last_acted_level` reset to -1 on response, and the gap model kept detecting "something to say."

### Root Cause
No suppression logic for HAS_PLAN state when all transitions are in the future.

### Fix
Added HAS_PLAN suppression: when the user has a plan and no transitions/bedtime checks are due, the `context` command returns `action_needed: null`. Silent monitoring until the next checkpoint arrives.

**Critical:** This suppression must exist in BOTH the CLI `context` command AND the enforcement script. If they disagree, you get either false escalation or false suppression.

### Lesson
**Having a plan IS engagement.** The user committed to a bedtime — that's the point. Don't undermine it by nagging them every 2 minutes about the plan they already made.

---

## 7. Threshold Boundary Jumps

**Severity: Low — but confusing when it happens**

### What Happened
At 10 PM (the threshold switch hour), the escalation level would jump because the system switched from relaxed to urgent thresholds based on current time.

### Example
User stopped responding at 9:50 PM. At 9:50 PM with relaxed thresholds (L1=10min), gap=10min → L1. At 10:01 PM, the system switches to urgent thresholds (L1=5min) and suddenly gap=11min → L2. The escalation jumped from L1 to L2 without the user's behavior changing.

### Fix
Pick threshold table based on when silence STARTED (`last_response_at`), not current time. If silence started at 9:50 PM, use relaxed thresholds for the entire gap regardless of what time it is now.

### Lesson
**Anchoring to the start of an event, not the current moment, prevents boundary discontinuities.** This applies anywhere you have time-based thresholds that change.

---

## 8. Call Duration Threshold

**Severity: High — caused false L4 escalation**

### What Happened
The L3→L4 advancement check used a 5-second duration threshold. The user answered two calls at 5 and 7 seconds respectively — both under threshold. The system advanced to L4 (calling their partner) because it considered L3 "unsuccessful."

### Root Cause
5 seconds is enough time to pick up and say "hello?" but not enough to listen to the message. The system interpreted a legitimate answer as a failure.

### Fix
Raised to 10 seconds. This means the person actually listened to at least part of the TTS message.

### Lesson
**Be generous with success criteria for phone calls.** People fumble with phones, say "hold on," or take a moment to process. 10 seconds is a reasonable floor for "they engaged with the call." Consider making this configurable per user.

---

## 9. Sleep Tracker Date Math

**Severity: Medium — morning reports showed wrong data**

### What Happened
Morning reports fetched the wrong night's data because of Oura's exclusive `end_date` and the sleep session's `day` field being the wake-up date, not the sleep date.

### Root Cause
Assumed `end_date` was inclusive (it's not) and assumed `day` = night of sleep (it's wake-up date).

### Fix
See `code-patterns.md` §Sleep Tracker Date Math for the correct queries. Test with edge cases: pre-midnight bedtime, post-midnight bedtime, naps.

### Lesson
**Every API has date semantics that will surprise you.** Read the docs carefully, test with boundary cases, and document the gotchas. Oura's exclusive end_date is the kind of thing that works 90% of the time and fails silently the rest.

---

## 10. Cron Spam

**Severity: Medium — user got duplicate messages**

### What Happened
The dispatcher sent the same message multiple times because it ran every 2 minutes and the `last_acted_level` wasn't being updated correctly.

### Root Cause
The topic session received the dispatch, sent the message, but didn't call `outreach_sent` or `escalation_acted` to mark the level as handled.

### Fix
The topic session MUST call the appropriate CLI command after sending each message:
- After first outreach → `outreach_sent`
- After L0 proactive message → `outreach_sent`
- After L1-L4 escalation message → `escalation_acted <level>`

The dispatcher then sees the updated `last_acted_level` and doesn't re-dispatch.

### Lesson
**Every action must have a corresponding state update.** If the system sends a message, it must mark that it sent it. Otherwise the next cron cycle will send it again.

---

## 11. Topic Session Has No Memory

**Severity: Low (UX issue, not correctness issue)**

### What Happened
The topic session agent has a limited context window. Over many nights of conversation, older context falls off. The agent doesn't remember past sleep coaching conversations, making it feel like starting fresh every night.

### Workaround
This is partially addressed by using state.json (which persists) and morning reports (which provide data). The agent doesn't need to remember conversations — it needs to know the current state and the data.

### Lesson
**Design for amnesia.** Assume the agent has no memory of past sessions. Put everything important in files (state, config, archives). The agent reads the files, not its own conversation history.

---

## 12. Grace Period Races

**Severity: Low — occasional double sends**

### What Happened
The enforcement cron ran while the topic session was mid-conversation. Both tried to handle the same escalation level.

### Fix
Added a `grace_period_seconds` (default 90s) to state. If `last_updated` is within the grace period, the enforcement script skips its run — the agent is probably handling it.

### Lesson
**When multiple actors share state, add a grace period.** It's a crude but effective way to prevent races without complex distributed locking.

---

## 13. First Message Delivery

**Severity: Medium — messages didn't arrive in the topic**

### What Happened
When the dispatcher sent context to the topic session via `sessions_send`, the topic session generated a response — but the first message in a session triggered by `sessions_send` needs to use the `message` tool explicitly to reach the Telegram/Discord topic.

### Root Cause
`sessions_send` puts context IN the session but doesn't trigger automatic delivery to the user's channel. The agent must use `message tool` (or equivalent) for the first outbound message.

### Fix
Agent instructions explicitly state: "After receiving a dispatch via sessions_send, send your message to the user using the message tool." After the user replies to that message, subsequent messages are delivered naturally through the channel.

### Lesson
**Understand the delivery pipeline of your messaging platform.** The first message in a session triggered by internal events (crons, session sends) may need explicit routing. Test this during setup.

---

## 14. Cron Agent Memory

**Severity: Medium — led to stale behavior**

### What Happened
Expected cron agents to remember things from previous runs. They don't. Each cron execution is a fresh agent with no memory of past executions.

### Fix
All stateful information must live in files. The cron prompt must explicitly instruct the agent to read the relevant files. Don't assume any cross-run memory.

### Lesson
**Crons are stateless by design.** Every piece of persistent state must be in a file, and every cron prompt must include instructions to read that file. The prompt IS the agent's entire brain for that run.

---

## Summary: The Top 3 Rules

1. **Call `response` on every user message.** This is the #1 failure mode across all versions. Make it impossible to forget.

2. **Derive, don't store.** Compute escalation level from the time gap. Don't maintain counters. Derived state can't drift.

3. **Test with dry-run before enabling phone calls.** Calling someone's partner at 12:28 AM is the kind of mistake you only make once.
