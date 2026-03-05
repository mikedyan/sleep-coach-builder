# 🐯 Sleep Coach Builder for OpenClaw

A complete blueprint for building a personalized AI sleep coaching system. Designed for [OpenClaw](https://github.com/openclaw/openclaw) agents.

## What Is This?

This isn't code you install — it's a **comprehensive reference document** that an OpenClaw agent reads and then builds the entire sleep coaching system from scratch, customized for the specific user.

The agent conducts a user interview to gather preferences, then implements everything tailored to that person's timezone, bedtime goal, messaging platform, escalation preferences, sleep tracker, and calling provider.

## What Gets Built

- **Conversational Sleep Coach** — AI agent that checks in each evening in a dedicated chat topic
- **State Machine** — Tracks nightly progress: INITIAL → ENGAGED → HAS_PLAN → IN_BED
- **Gap-Based Escalation** — If the user goes silent, nudges ramp up over time
- **Phone Call Escalation** (optional) — Call the user or their accountability partner
- **Sleep Tracker Integration** (optional) — Morning reports with Oura, Whoop, Fitbit, Garmin, or Apple Watch data
- **Bedtime Tracker** — Long-term success tracking with visualizations

## How to Use

### As an OpenClaw Skill

```bash
# Install the skill
clawhub install sleep-coach

# Or manually: copy the sleep-coach/ folder to ~/.openclaw/skills/
```

Then tell your OpenClaw agent: *"I want to set up a sleep coaching system"* — it will read the skill and walk you through the setup.

### As a Reference

Even without OpenClaw, the documents are valuable reading for anyone building a similar system:

- **[SKILL.md](SKILL.md)** — Overview and entry point
- **[references/implementation-guide.md](references/implementation-guide.md)** — Full setup flow with user interview protocol
- **[references/architecture.md](references/architecture.md)** — State machine, gap-based escalation model, dispatcher pattern
- **[references/code-patterns.md](references/code-patterns.md)** — Critical algorithms (file locking, atomic writes, date math)
- **[references/lessons-learned.md](references/lessons-learned.md)** — 14 hard-won lessons from 5 versions of development

## International Support

The system isn't US-only. The user interview covers:

- **Telephony**: Twilio, Plivo, Telnyx, Vonage, Sinch, Exotel (by region)
- **TTS**: ElevenLabs, OpenAI, Google Cloud, Deepgram, Chatterbox (open source)
- **Sleep Trackers**: Oura, Whoop, Fitbit, Garmin, Apple Watch, or manual self-report
- **Messaging**: Telegram, Discord, Slack, WhatsApp, Signal

## Philosophy

The system is built on **conversational accountability, not punishment**. The metaphor is "effortless like breathing" — help the person *want* to go to bed on time through understanding and gentle nudges, not shame or restrictions.

## Origin

Built through 5 iterations (v1-v5) over January-February 2026 as a personal sleep coaching system. The lessons-learned document captures every failure and fix from real-world usage, including a memorable incident involving a false phone call to an accountability partner at 12:28 AM. 😬

## License

MIT
