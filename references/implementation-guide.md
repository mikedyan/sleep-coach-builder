# Implementation Guide

Complete step-by-step process for building a personalized sleep coaching system.

## Table of Contents

1. [User Interview](#1-user-interview)
2. [Component Selection](#2-component-selection)
3. [Build Order](#3-build-order)
4. [Step 1: Configuration File](#step-1-configuration-file)
5. [Step 2: State Management](#step-2-state-management)
6. [Step 3: Main CLI](#step-3-main-cli)
7. [Step 4: Enforcement Module](#step-4-enforcement-module)
8. [Step 5: Morning Report](#step-5-morning-report)
9. [Step 6: Bedtime Tracker](#step-6-bedtime-tracker)
10. [Step 7: Agent Instructions](#step-7-agent-instructions)
11. [Step 8: Cron Jobs](#step-8-cron-jobs)
12. [Step 9: Testing & Go-Live](#step-9-testing--go-live)

---

## 1. User Interview

Conduct this interview before writing any code. The answers determine which components to build and how to configure them.

### Phase 1: Goals & Philosophy

Ask these questions conversationally, not as a form:

**Bedtime Target**
- "What time do you want to be in bed by?" (Not asleep — just in bed, screens off)
- "Is this a hard goal or aspirational? Should we track a slightly looser time as 'success'?"
- Recommend: set success threshold 15-30 min after their ideal (e.g., goal=11 PM, success=before 11:30 PM)

**Current Habits**
- "What time do you usually end up going to bed?"
- "Is it consistent or does it vary a lot?"
- "What typically keeps you up? (work, phone, TV, kids, partner)"

**Motivation**
- "Why do you want to fix this? Health, productivity, family, energy?"
- This informs the coaching tone — a parent wanting energy for their kids gets different messaging than a founder optimizing performance

**Coach Assertiveness**
- "How pushy should I be? Options:"
  - 🟢 **Gentle**: Friendly reminders only, no escalation
  - 🟡 **Moderate**: Persistent messages that ramp up in urgency
  - 🔴 **Aggressive**: Messages → phone calls → contact someone else
- "Would you want me to call you if you're not responding?"
- "Is there someone (partner, roommate, friend) who could nudge you as a last resort?"

**Philosophy Check**
- "Should consequences feel punishing or reflective? (We recommend reflective — conversations about what happened, not shame)"
- Explain: The system works best when it builds internal motivation, not fear

### Phase 2: Technical Setup

**Timezone**
- "What timezone are you in?" (Need IANA format, e.g., `America/New_York`, `Europe/London`)

**Messaging Platform**
- "Which platform is your OpenClaw connected to?" (Telegram, Discord, Slack, etc.)
- "Do you want the sleep coach in a dedicated topic/thread/channel, or in the main chat?"
- Recommend: Dedicated topic/thread — keeps sleep conversations separate and builds conversation history
- Note which platform features are available (topics, threads, reactions, etc.)

**Sleep Tracker**
- "Do you use a sleep tracking device? Which one?"
  - **Oura Ring** — Best API for sleep data. REST API with personal access token. Provides: bedtime_start, sleep stages, HRV, sleep score, efficiency. Get token at cloud.ouraring.com → Personal Access Tokens.
  - **Whoop** — Good API. OAuth2 flow. Provides: sleep performance, strain, recovery, HRV. Developer app at developer.whoop.com.
  - **Fitbit / Google** — Web API. OAuth2. Provides: sleep stages, duration, sleep score. Setup at dev.fitbit.com.
  - **Garmin** — Connect API. OAuth1. Provides: sleep stages, pulse ox, body battery. Apply at developer.garmin.com.
  - **Apple Watch** — No direct cloud API. Options: (a) Use Health Auto Export app to push to a server, (b) Use Apple HealthKit via Shortcuts, (c) Use a third-party aggregator like Spike API or Open Wearables.
  - **Samsung / Google Health Connect** — Similar to Apple Watch, no direct cloud API. Use Health Connect API on Android or third-party aggregators.
  - **No tracker** — System works without one. Morning reports will rely on self-reported data. Ask: "Would you be willing to tell me each morning what time you went to bed?"
- If they have a tracker, confirm API access and ask them to generate a token/key

**Accountability Partner** (if assertiveness is Moderate or Aggressive)
- "Who should I contact as a last resort? Name, phone number, and relationship"
- "What should I say to them? Something like 'Hey [name], [user] asked me to nudge them about bedtime but they're not responding — could you check on them?'"
- "Are there times I should NEVER contact them? (e.g., after midnight, weekdays only)"

### Phase 3: Escalation Configuration

Based on assertiveness level, configure escalation tiers:

**For Gentle (🟢):**
- L1: Friendly Telegram/Discord message — that's it
- No phone calls, no partner contact
- Thresholds: Single reminder, maybe a follow-up after 20 min

**For Moderate (🟡):**
- L1 (0 min): Friendly check-in message
- L2 (+15 min no response): More urgent message
- L3 (+30 min no response): Final message with gentle guilt
- No phone calls
- Thresholds configurable

**For Aggressive (🔴):**
- L1 (0 min): Friendly check-in
- L2 (+10-15 min): Urgent message
- L3 (+20-30 min): Phone call to user
- L4 (+30-45 min): Contact accountability partner
- L5 (optional): Smart home actions (lights, speakers)

**Time-based threshold adjustment:**
- "Should escalation be faster as it gets later? (e.g., more patient at 9 PM, more urgent at 11 PM)"
- Recommend: Yes. Use two threshold tables:
  - Before target bedtime: relaxed intervals (10-15 min between levels)
  - After target bedtime: urgent intervals (5-10 min between levels)

**Active Hours Window:**
- "When should the system start checking in?" (Recommend: 30-60 min before target wind-down)
- "When should it give up for the night?" (Recommend: 3-4 AM — hard deadline)
- Handle midnight crossover: active from e.g. 9:15 PM to 4:00 AM spans two calendar days

### Phase 4: Integration Details

Based on Phase 2-3 answers, gather specific credentials:

**If phone calls enabled:**
- "Which country are you in?" Then recommend a telephony provider:
  - **US/Canada**: Twilio (most popular, well-documented, ~$0.014/min outbound)
  - **Europe**: Twilio, Vonage, or Sinch (good EU presence, GDPR-aware)
  - **UK**: Twilio or Vonage (strong UK number support)
  - **India/South Asia**: Exotel (local carrier expertise) or Plivo (190 countries, cheaper)
  - **Australia/NZ**: Twilio or Plivo
  - **Global/Other**: Plivo (190 countries, simple API) or Telnyx (competitive international rates)
  - **Budget-conscious**: Telnyx or Plivo (generally 30-50% cheaper than Twilio)
- Guide them through account setup and getting a phone number
- Get: Account SID/API key, Auth token, Phone number
- Store credentials securely (1Password, environment variables, or encrypted config)

**If phone calls enabled — TTS provider:**
- "For phone calls, I need to convert text to speech. Options:"
  - **ElevenLabs** — Best quality, natural voices, 32 languages. ~$0.30/1K chars. Good for: premium feel, multilingual users.
  - **OpenAI TTS** — Good quality, simple API, 6 voices. ~$0.015/1K chars. Good for: English speakers, cost-sensitive.
  - **Google Cloud TTS** — 220+ voices, 40+ languages, WaveNet voices sound natural. ~$0.016/1K chars (WaveNet). Good for: non-English speakers, budget-conscious.
  - **Deepgram** — Low latency, good for real-time. Good for: developers who want speed.
  - **Built-in Twilio TTS** — Free with Twilio calls, robotic but functional. Good for: simplest setup, no extra cost.
  - **Chatterbox (open source)** — Self-hosted, free, surprisingly good. 23 languages. Good for: privacy-focused, tech-savvy users.
- Get API key for chosen provider

**If sleep tracker enabled:**
- Walk through API token/key generation for their specific tracker
- Test the API with a simple request to verify it works
- Identify which fields their tracker provides (bedtime_start, sleep_score, HRV, etc.)

**Smart Home (optional L5):**
- "Do you have smart speakers or lights you'd want me to use?"
- Options: Sonos API, Home Assistant, Philips Hue, Google Home (via routines)
- This is the most variable — implementation depends entirely on their setup

---

## 2. Component Selection

Based on the interview, determine which components to build:

| Component | Required? | Depends On |
|---|---|---|
| Config file | Always | Interview answers |
| State management (lib) | Always | — |
| Main CLI | Always | State management |
| Agent instructions | Always | Platform choice |
| Dispatcher cron | Always | CLI + messaging platform |
| Enforcement module | If phone calls enabled | Telephony + TTS provider |
| Enforcement cron | If phone calls enabled | Enforcement module |
| Morning report | If sleep tracker available | Tracker API |
| Morning report cron | If morning report built | Morning report |
| Bedtime tracker | If sleep tracker available | Tracker API |
| Tracker update cron | If tracker built | Bedtime tracker |
| Visualization | If tracker built | Tracker data |

---

## 3. Build Order

Build in this order — each step depends on the previous:

1. **Config file** — Central configuration, referenced by everything
2. **State management library** — File locking, state schema, escalation derivation
3. **Main CLI** — State machine interface (context, response, plan, in_bed, etc.)
4. **Enforcement module** — Phone calls (skip if not enabled)
5. **Morning report** — Sleep tracker data fetcher (skip if no tracker)
6. **Bedtime tracker** — Long-term success tracking (skip if no tracker)
7. **Agent instructions** — How the topic session agent should behave
8. **Cron jobs** — Wire up automation
9. **Testing** — Dry-run everything before going live

See `architecture.md` for the state machine design and `code-patterns.md` for critical algorithms.

---

## Step 1: Configuration File

Create `config.json` in the sleep coach directory with all user-specific settings:

```json
{
  "user": {
    "name": "<user's first name>",
    "timezone": "<IANA timezone>"
  },
  "goals": {
    "target_bedtime": "23:00",
    "success_threshold": "23:30",
    "active_start": "21:15",
    "hard_deadline": "04:00",
    "reset_hour": 4
  },
  "escalation": {
    "assertiveness": "aggressive|moderate|gentle",
    "thresholds_relaxed_min": [10, 20, 30, 40],
    "thresholds_urgent_min": [5, 10, 15, 20],
    "threshold_switch_hour": 22,
    "grace_period_seconds": 90
  },
  "contacts": {
    "user_phone": "<phone with country code>",
    "user_messaging_id": "<telegram/discord ID>",
    "partner_phone": "<if applicable>",
    "partner_name": "<if applicable>",
    "partner_contact_rules": {
      "not_before": "08:00",
      "not_after": "01:00",
      "days": ["mon","tue","wed","thu","fri","sat","sun"]
    }
  },
  "telephony": {
    "enabled": false,
    "provider": "twilio|plivo|telnyx|vonage",
    "from_number": "<outbound number>",
    "credentials": "<reference to secure storage>"
  },
  "tts": {
    "provider": "elevenlabs|openai|google|twilio_builtin|none",
    "voice_id": "<provider-specific>",
    "model": "<provider-specific>",
    "language": "<ISO code>"
  },
  "sleep_tracker": {
    "enabled": false,
    "provider": "oura|whoop|fitbit|garmin|manual",
    "morning_report_time": "08:00",
    "credentials": "<reference to secure storage>"
  },
  "smart_home": {
    "enabled": false,
    "provider": "<sonos|homeassistant|hue>",
    "config": {}
  },
  "philosophy": {
    "consequence_type": "reflective_conversation",
    "tone": "warm but direct"
  }
}
```

Adapt the schema based on what the user chose to enable. Don't include sections for disabled features.

---

## Step 2: State Management

Build the state management library first. This is the foundation everything else depends on.

See `architecture.md` §State Schema for the full schema and `code-patterns.md` for implementation details.

**Key requirements:**
- File locking (fcntl on Unix, msvcrt on Windows) for concurrent access from crons + topic session
- Atomic writes (write to temp file, then rename) to prevent corruption
- Auto-reset after the configured reset hour (default 4 AM)
- State archiving before reset (for historical records)
- Timezone-aware datetime handling throughout
- Midnight crossover support (active hours span two calendar days)

**The state file tracks:**
- Current state machine position (INITIAL/ENGAGED/HAS_PLAN/IN_BED)
- Escalation timing (last_response_at, last_acted_level)
- The user's bedtime plan (summary, expected_bedtime, transition checkpoints)
- Call records (for deduplication — don't call twice in one night)
- Session metadata (start time, end time, success)

---

## Step 3: Main CLI

Build a CLI tool that is the **only interface for state mutations**. This is critical — all state changes go through this tool, whether called by the agent, crons, or enforcement scripts.

See `architecture.md` §CLI Commands for the full command list.

**Essential commands:**
- `context` — Returns current state + derived escalation level + what action is needed (JSON)
- `response` — Register that the user responded (resets escalation timer)
- `plan_json <json>` — Store the user's bedtime plan with transition checkpoints
- `outreach_sent` — Mark that a message was sent (starts the escalation clock)
- `escalation_acted <level>` — Mark that an escalation level was acted on
- `in_bed [text]` — Mark user as in bed, end session
- `status` — Human-readable status dump

**The `context` command is the brain.** It computes:
1. What the current escalation level is (derived from time gap, not stored)
2. Whether any plan transitions are due
3. What action the agent should take next
4. Why (human-readable reason)

The dispatcher cron calls `context` every 2 minutes and acts on the result.

---

## Step 4: Enforcement Module

Skip if phone calls are not enabled.

Build a separate enforcement script that handles phone calls. It runs as its own cron, separate from the dispatcher, for safety isolation.

**Flow:**
1. Load state, derive escalation level from gap
2. If level ≥ 3 and L3 call not yet attempted → call user
3. If level ≥ 4 and L3 was unsuccessful and L4 not yet attempted → call partner
4. Update state with call results (SID, status, duration)

**Call flow:**
1. Generate message text (dynamic based on time of day and context)
2. Convert to audio via TTS provider
3. Upload audio to temporary hosting (litterbox.catbox.moe, tmpfiles.org, or similar)
4. Initiate call via telephony provider with audio URL
5. Wait ~35 seconds, then check call status
6. Record result in state for deduplication

**Call success criteria:** Status = "completed" AND duration > 10 seconds. Short calls (5-7 seconds) often mean the person picked up and immediately hung up — that's not a real conversation. Use a generous threshold.

**Safety: Respect partner contact rules** from config (not before/after certain hours, specific days only).

**Telephony provider patterns:**
- Twilio: POST to `/Calls.json` with TwiML `<Response><Play>url</Play></Response>`
- Plivo: POST to `/Call/` with answer_url pointing to XML
- Vonage: POST to `/calls` with NCCO actions
- Telnyx: POST to `/calls` with TeXML
- Each has similar concepts — adapt the HTTP calls to the provider's API

---

## Step 5: Morning Report

Skip if no sleep tracker is available. If using manual self-report, build a simplified version that just asks "What time did you go to bed last night?"

**Flow:**
1. Fetch last night's sleep data from the tracker API
2. Load last night's bedtime plan from state archive
3. Compare actual vs. planned bedtime (accountability)
4. Format into a structured report
5. Output as JSON for the topic session agent to narrate

**Key data points to extract (varies by tracker):**
- Bedtime start time (SOURCE OF TRUTH for accountability)
- Total sleep duration
- Deep sleep / REM sleep duration
- Sleep efficiency (time asleep / time in bed)
- Sleep score (composite metric)
- Heart rate / HRV (recovery indicators)
- Wake-up time

**Sleep tracker API gotchas:**
- Each tracker has its own date semantics. See `code-patterns.md` for details.
- Oura's `end_date` parameter is EXCLUSIVE — to get data for day X, query `end_date=X+1`
- Most trackers assign the sleep session to the WAKE-UP date, not the go-to-bed date
- Always convert bedtime_start to local timezone before determining which "evening" it belongs to
- If bedtime is after midnight but before 4 AM, it belongs to the previous evening

**Accountability comparison:**
```
actual_bedtime vs expected_bedtime:
  diff ≤ 15 min → WIN
  diff > 15 min late → LATE
  diff > 15 min early → EARLY (still good!)
```

---

## Step 6: Bedtime Tracker

Skip if no sleep tracker available (can't verify bedtimes without one).

Build a long-term tracking system that scores each night and computes trends.

**Data structure:**
```json
{
  "target": "23:30",
  "records": {
    "2026-01-15": {
      "bedtime_start": "2026-01-15T23:08:00-08:00",
      "bedtime_time": "23:08",
      "success": true,
      "sleep_score": 87,
      "total_sleep_min": 460
    }
  },
  "last_updated": "2026-01-16T09:30:00-08:00"
}
```

**Features:**
- Backfill: Fetch historical data to establish a baseline
- Daily update: Cron adds last night's data each morning
- Stats: Success rate, current streak, longest streak, monthly/weekly breakdowns
- Visualization: Generate a chart showing bedtime distribution, weekly success rates, and calendar heatmap

**Visualization (optional but motivating):**
Use matplotlib or build an HTML chart. Key views:
- Scatter plot: bedtime times over date range (green = success, red = miss)
- Target line showing the cutoff
- Weekly bar chart of success percentage
- Monthly summary with rates
- Calendar heatmap (green/red cells)

---

## Step 7: Agent Instructions

Write instructions for the topic session agent. This is the AI that lives in the dedicated sleep coach topic/thread and has conversations with the user.

**The agent needs to know:**

1. **Identity**: "You are the Sleep Coach for [name]. Your goal is to help them get to bed by [time]."

2. **Mandatory state updates**: Every time the user sends a message during active hours, the agent MUST call the CLI's `response` command. This resets the escalation timer. If the agent fails to do this, escalation will fire incorrectly.

3. **Conversation flow by state:**
   - INITIAL: Greet, ask about evening plans, what's on the agenda
   - ENGAGED: Push for a bedtime plan — "What's your plan for tonight? When are you aiming for bed?"
   - HAS_PLAN: Monitor plan transitions, check in at scheduled times
   - IN_BED: Say goodnight, session complete

4. **Plan creation**: When the user describes their plan, the agent generates a `plan_json` call with:
   - Summary of the plan
   - Expected bedtime (ISO timestamp)
   - Transition checkpoints — moments to check in (e.g., "movie ends at 10:45 PM")

5. **Escalation handling**: The agent sends L1-L2 messages when dispatched. Tone should match the configured assertiveness:
   - L1: "Hey, haven't heard from you in a bit — everything okay?"
   - L2: "I'm getting a little worried. Can you send me a quick message?"
   - L3-L4 are handled by the enforcement module, not the agent

6. **Morning reports**: When the morning report cron triggers, the agent narrates the data conversationally — not as a raw data dump. Highlight wins, note areas for improvement, compare to plan.

7. **Tone**: Match the user's preferred coaching style. Warm and supportive, not clinical. Use the user's name. Keep messages short — this is a chat, not an essay.

---

## Step 8: Cron Jobs

Set up OpenClaw cron jobs to automate the system.

### Dispatcher Cron (Required)

The dispatcher runs every 2 minutes during active hours, checks state, and tells the topic session what to do.

**Schedule:** `*/2 <active_hours> * * *`
Example for 9 PM - 4 AM: `*/2 21-23,0-3 * * *`

**Model:** Use a cost-effective model (Sonnet-class). The dispatcher is simple logic, not creative work.

**Prompt pattern:**
```
1. Run the CLI: python3 <path>/sleep_coach.py context
2. Read the JSON output
3. If action_needed is null → reply NO_REPLY (nothing to do)
4. If action_needed is not null → use sessions_send to send the context
   to the topic session, including action_needed and action_reason
5. If action is escalation_L1 or L2 → also tell the topic session what
   to say and to call escalation_acted <level> after sending
6. Reply NO_REPLY
```

**Critical**: The dispatcher is intentionally dumb. It checks state and delegates. The topic session (running a smarter model) handles all conversation and decision-making.

### Enforcement Cron (If phone calls enabled)

Separate cron for phone call safety.

**Schedule:** Same as dispatcher: `*/2 <active_hours> * * *`
**Model:** Cost-effective model
**Prompt:** `Run python3 <path>/enforce.py. Reply NO_REPLY.`

Why separate? Phone calls are high-stakes. Isolating them in their own cron with its own script means the dispatcher can't accidentally trigger calls through prompt confusion.

### Morning Report Cron (If sleep tracker enabled)

**Schedule:** `0 <morning_hour> * * *` (e.g., `0 8 * * *`)
**Prompt:**
```
1. Run python3 <path>/morning_data.py
2. Read the JSON output
3. Use sessions_send to send the data to the topic session
4. Include instructions: "Narrate this morning report conversationally.
   Highlight wins, note concerns, compare to plan if available."
5. Reply NO_REPLY
```

### Bedtime Tracker Update Cron (If tracker built)

**Schedule:** `30 <morning_hour> * * *` (e.g., `30 9 * * *` — 30 min after morning report)
**Prompt:** `Run python3 <path>/bedtime_tracker.py. Reply NO_REPLY.`

---

## Step 9: Testing & Go-Live

### Pre-Launch Checklist

Run through each test manually:

- [ ] **CLI context**: Run `sleep_coach.py context` — should return INITIAL state with first_outreach action
- [ ] **CLI response**: Run `sleep_coach.py response` — should transition to ENGAGED
- [ ] **CLI plan_json**: Set a test plan — should transition to HAS_PLAN
- [ ] **CLI in_bed**: Mark in bed — should transition to IN_BED
- [ ] **State reset**: Change the date in state.json to yesterday, run context — should archive and reset
- [ ] **Escalation derivation**: Set last_response_at to 20 min ago, run context — should show L1 or L2
- [ ] **Morning report** (if enabled): Run morning_data.py — should return JSON with sleep data
- [ ] **Enforcement dry run** (if enabled): Run `SLEEP_COACH_DRY_RUN=true python3 enforce.py` — should log actions without calling
- [ ] **Bedtime tracker** (if enabled): Run with --stats — should show summary
- [ ] **Cron test**: Use OpenClaw's `schedule.kind: "at"` to fire a one-shot dispatcher test

### Dry Run Mode

Implement a `DRY_RUN` environment variable that:
- Logs what would happen without actually calling phones
- Prints messages that would be sent
- Skips external API calls
- Still updates state (so you can test the state machine flow)

### Go-Live

1. Enable the dispatcher cron first (messaging only)
2. Run for 2-3 nights with just L1-L2 messaging
3. If enforcement is enabled, add the enforcement cron after confirming messaging works
4. Monitor the enforcement.log for any issues
5. After a week of clean runs, consider the system stable

### Monitoring

- Check `enforcement.log` periodically for errors
- Review state archives to verify nightly sessions are working
- Watch for the classic failure: topic session not calling `response` on user messages (see lessons-learned.md)

---

## Maintenance

### Regular Tasks
- Review and update escalation thresholds if they feel too aggressive or too passive
- Update TTS messages periodically so they don't feel robotic and repetitive
- If the user's schedule changes (new timezone, new bedtime goal), update config.json

### State Hygiene
- State archives accumulate daily — consider a cleanup cron for archives older than 90 days
- The bedtime tracker JSON grows indefinitely — this is fine, it's small data

### Evolving the System
- After 2-4 weeks, review the bedtime tracker data with the user
- Ask: "Is this working? Too aggressive? Not enough? What would make it better?"
- Adjust thresholds, tone, and features based on feedback
- The system should evolve with the user's habits
