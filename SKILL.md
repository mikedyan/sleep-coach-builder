---
name: sleep-coach-builder
description: >
  Build a personalized AI sleep coaching system with bedtime accountability,
  multi-channel escalation, sleep tracker integration, and morning reports.
  Use when: (1) someone wants help getting to bed on time, (2) wants a bedtime
  accountability system, (3) asks about sleep coaching or tracking, (4) wants to
  improve sleep habits with AI assistance. Conducts a thorough user interview to
  customize the entire system for the individual — timezone, bedtime goal,
  messaging platform, escalation preferences (messages → phone calls →
  accountability partner), sleep tracker (Oura/Whoop/Fitbit/Garmin/Apple Watch
  or none), and calling provider. Works internationally.
---

# Sleep Coach

Build a complete AI sleep coaching system through conversational accountability.

## Philosophy

**Not punishment-based.** The system grows internal motivation through understanding and gentle accountability. The metaphor is "effortless like breathing" — help the person *want* to go to bed, don't force them.

## What Gets Built

A modular system with these components (user chooses which to enable):

1. **State Machine** — Tracks nightly progress: INITIAL → ENGAGED → HAS_PLAN → IN_BED
2. **Conversational Coach** — AI agent in a dedicated chat topic/thread that checks in each evening
3. **Gap-Based Escalation** — If the user goes silent, escalation ramps up over time (configurable)
4. **Sleep Tracker Integration** — Morning reports with objective data (Oura, Whoop, Fitbit, etc.)
5. **Bedtime Tracker** — Long-term success tracking with visualizations
6. **Phone Call Escalation** — Optional: call the user or their accountability partner via TTS + telephony

## Setup Flow

1. Read `references/implementation-guide.md` — the full step-by-step process
2. Conduct the **User Interview** (Phase 1–4) to gather all requirements
3. Build components based on what the user chose
4. Set up cron jobs for automation
5. Test with dry-run mode before going live

## Reference Documents

Read these as needed during implementation:

- **`references/implementation-guide.md`** — Start here. Full setup flow, user interview protocol, step-by-step build instructions, cron design, testing checklist.
- **`references/architecture.md`** — Deep dive into the state machine, gap-based escalation model, component interactions, and state schema. Read when building the state management and CLI.
- **`references/code-patterns.md`** — Critical algorithms: file locking, atomic writes, escalation derivation, midnight crossover, sleep tracker API date math. Read when writing the actual code.
- **`references/lessons-learned.md`** — Hard-won lessons from 5 versions of development. Every major failure and its fix. **Read before going live** — these mistakes are easy to repeat.

## Minimum Viable System

Not everything is required. The simplest version is:
- State machine + CLI (Python, ~300 lines)
- Agent instructions for the topic session
- One dispatcher cron (every 2 min during active hours)
- No sleep tracker, no phone calls, no visualization

Everything else layers on top.
