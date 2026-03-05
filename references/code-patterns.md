# Code Patterns

Critical algorithms and patterns that must be implemented correctly. These are the pieces where getting it wrong causes subtle, hard-to-debug failures.

## Table of Contents

1. [File Locking](#file-locking)
2. [Atomic Writes](#atomic-writes)
3. [Gap-Based Escalation Derivation](#gap-based-escalation-derivation)
4. [Midnight Crossover](#midnight-crossover)
5. [Sleep Tracker Date Math](#sleep-tracker-date-math)
6. [Phone Call Flow](#phone-call-flow)
7. [State Reset Logic](#state-reset-logic)
8. [Response Registration](#response-registration)
9. [HAS_PLAN Suppression](#has_plan-suppression)
10. [Timezone Handling](#timezone-handling)

---

## File Locking

Multiple processes access state.json simultaneously: the topic session agent, dispatcher cron, enforcement cron. Without locking, concurrent writes corrupt the file.

**Pattern: fcntl exclusive lock via a separate .lock file**

```python
import fcntl
from contextlib import contextmanager

LOCK_FILE = STATE_FILE.with_suffix('.lock')

@contextmanager
def locked_state():
    LOCK_FILE.touch(exist_ok=True)
    with open(LOCK_FILE, 'r') as lock:
        fcntl.flock(lock.fileno(), fcntl.LOCK_EX)
        try:
            yield
        finally:
            fcntl.flock(lock.fileno(), fcntl.LOCK_UN)
```

**Why a separate .lock file?** Locking the state file directly is fragile — if you open it for writing and lock simultaneously, truncation can happen before the lock is acquired.

**Windows alternative:** Use `msvcrt.locking()` with `LK_LOCK`. Or use `filelock` pip package for cross-platform.

---

## Atomic Writes

Never write directly to state.json. A crash mid-write produces a corrupted or empty file.

**Pattern: Write to temp file, then rename**

```python
import json
from pathlib import Path

def save_state_unlocked(state: dict):
    temp_file = STATE_FILE.with_suffix('.tmp')
    temp_file.write_text(json.dumps(state, indent=2))
    temp_file.rename(STATE_FILE)  # atomic on same filesystem
```

`rename()` is atomic on POSIX filesystems — the file is either the old version or the new version, never a partial write.

**Combine with locking:** Always acquire the lock before calling save.

---

## Gap-Based Escalation Derivation

The core escalation algorithm. Derive the escalation level from the time gap, never store it.

```python
from datetime import datetime
from zoneinfo import ZoneInfo

def derive_escalation_level(state: dict) -> tuple[int, float]:
    """Returns (level, gap_minutes).
    
    If last_response_at is None, returns (0, 0.0) — guard against
    premature escalation before first contact.
    """
    last_response_str = state["escalation"].get("last_response_at")
    if not last_response_str:
        return (0, 0.0)

    last_response = datetime.fromisoformat(last_response_str)
    if last_response.tzinfo is None:
        last_response = last_response.replace(tzinfo=TZ)

    gap = datetime.now(TZ) - last_response
    gap_minutes = gap.total_seconds() / 60.0

    # Pick thresholds based on when silence STARTED
    thresholds = get_thresholds_for_time(last_response)

    level = 0
    for i, threshold in enumerate(thresholds):
        if gap_minutes >= threshold:
            level = i + 1
        else:
            break

    return (level, gap_minutes)
```

**Critical detail:** The threshold table is chosen based on `last_response_at`, NOT current time. If silence started at 9:50 PM (relaxed), use relaxed thresholds for the entire gap, even when current time passes the switch hour. This prevents sudden level jumps at the boundary.

---

## Midnight Crossover

Active hours span midnight (e.g., 9:15 PM to 4:00 AM). Standard time comparisons break.

```python
def is_active_hours(state: dict) -> bool:
    current = datetime.now(TZ)
    
    active_start = parse_time(state["config"]["active_start"], current)
    hard_deadline = parse_time(state["config"]["hard_deadline"], current)

    # Crossover case: deadline is "earlier" than start (spans midnight)
    if hard_deadline <= active_start:
        return current >= active_start or current < hard_deadline

    # Normal case: both on same day
    return active_start <= current < hard_deadline
```

The key insight: when the deadline time is numerically less than the start time (4:00 < 21:15), the window crosses midnight. In that case, use OR logic — either after start OR before deadline.

---

## Sleep Tracker Date Math

Each tracker has its own date semantics. Getting this wrong means pulling the wrong night's data.

### Oura Ring

**Oura's `end_date` is EXCLUSIVE.** To get data for February 10:
```
start_date=2026-02-10&end_date=2026-02-11
```

**Oura's `day` field** = the date you WOKE UP, not the date you went to sleep.

**Mapping sleep sessions to evenings:**
```python
def get_evening_date(bedtime_start_iso: str) -> str:
    """Convert a bedtime_start timestamp to the 'evening date' it belongs to.
    
    If bedtime is after midnight but before 4 AM,
    it belongs to the previous calendar day's evening.
    """
    dt = datetime.fromisoformat(bedtime_start_iso).astimezone(TZ)
    if dt.hour < 4:
        evening = dt.date() - timedelta(days=1)
    else:
        evening = dt.date()
    return evening.isoformat()
```

**Fetching "last night's" data at 8 AM:**
```python
today = datetime.now(TZ).strftime('%Y-%m-%d')
yesterday = (datetime.now(TZ) - timedelta(days=1)).strftime('%Y-%m-%d')
tomorrow = (datetime.now(TZ) + timedelta(days=1)).strftime('%Y-%m-%d')

# Query both yesterday and today to catch sessions tagged either day
sessions = oura_get('sleep', {
    'start_date': yesterday,
    'end_date': tomorrow  # exclusive, so this covers today
})

# Find the long_sleep session for today (wake-up date)
long_sleep = next(
    (s for s in sessions['data']
     if s['type'] == 'long_sleep' and s['day'] == today),
    None
)
```

### Whoop

Whoop's sleep API uses `start` and `end` datetime parameters. Sleep records have a `score.sleepNeeded`, `score.sleepPerformancePercentage`, etc. The `start` field is when sleep started (equivalent to Oura's `bedtime_start`).

### Fitbit

Fitbit's sleep log endpoint takes a single `date` parameter (the date you want sleep FOR). It returns sessions with `startTime` and `endTime`. Date handling is simpler.

### General Rule

Always test with edge cases:
- Sleep that starts before midnight and ends after (the common case)
- Sleep that starts after midnight (went to bed at 1 AM)
- Naps vs. main sleep (filter for main/long sleep sessions)
- Missing data (tracker wasn't worn, battery died)

---

## Phone Call Flow

The sequence for making an escalation phone call:

```
1. Generate message text (time-aware, context-aware)
2. Convert text → audio (TTS provider API)
3. Upload audio → temporary URL
4. Initiate call with audio URL (telephony provider)
5. Wait ~35 seconds
6. Check call status
7. Record result in state
```

### TTS Generation Pattern

```python
def generate_audio(text: str, provider: str, config: dict) -> bytes:
    if provider == "elevenlabs":
        # POST to /v1/text-to-speech/{voice_id}
        # Returns audio bytes directly
        ...
    elif provider == "openai":
        # POST to /v1/audio/speech
        # model, voice, input
        ...
    elif provider == "google":
        # POST to texttospeech.googleapis.com/v1/text:synthesize
        # Returns base64 audio
        ...
    elif provider == "twilio_builtin":
        return None  # Use TwiML <Say> instead of <Play>
```

### Audio Hosting

TTS generates audio bytes. Telephony providers need a URL to play. Options:
- **litterbox.catbox.moe** — Free, temporary (1h-72h), no auth needed. Simple curl upload.
- **tmpfiles.org** — Free, temporary. Similar to litterbox.
- **S3/GCS/R2** — If available. More reliable but requires setup.
- **Self-hosted** — If the OpenClaw host has a public URL.

Pattern for litterbox:
```python
result = subprocess.run(
    ['curl', '-s', '-F', 'reqtype=fileupload', '-F', 'time=1h',
     '-F', f'fileToUpload=@{temp_audio_path}',
     'https://litterbox.catbox.moe/resources/internals/api.php'],
    capture_output=True, text=True, timeout=60
)
audio_url = result.stdout.strip()
```

### Call Success Criteria

After initiating the call, wait ~35 seconds, then check status:

```python
status, duration = get_call_status(call_sid)

# A call is "successful" if:
# 1. Status is "completed" (not "no-answer", "busy", "failed")
# 2. Duration > 10 seconds (short pickups aren't real conversations)
success = status == "completed" and duration > 10
```

**Why 10 seconds?** Early versions used 5 seconds, but people who pick up and immediately say "what?" before hanging up at 5-7 seconds aren't engaging with the message. 10 seconds means they listened.

### Call Deduplication

L3 and L4 calls each use an `attempted_at` timestamp as a dedup key:
- `attempted_at = null` → call not yet made tonight → eligible
- `attempted_at = <timestamp>` → already called → skip

This resets daily with the state reset. One L3 and one L4 call per night, max.

---

## State Reset Logic

The state resets daily for a fresh start. The tricky part is handling the midnight crossover.

```python
def should_reset(state: dict) -> bool:
    # Same date → no reset
    if state['date'] == today_str():
        return False
    
    # Different date BUT before reset hour → don't reset yet
    # (session from last night is still active)
    if datetime.now(TZ).hour < state['config']['reset_hour']:
        return False
    
    return True
```

**Timeline:**
```
11 PM (Feb 10) — Session active, state.date = "2026-02-10"
 1 AM (Feb 11) — state.date ≠ today, but hour < 4 → NO reset, session continues
 4 AM (Feb 11) — state.date ≠ today and hour ≥ 4 → RESET, archive old state
 ... (state dormant during the day)
 9:15 PM (Feb 11) — New session starts, state.date = "2026-02-11"
```

**Always archive before resetting:** Save the old state to `state-archive/YYYY-MM-DD.json` for historical reference and morning report access.

---

## Response Registration

The most critical pattern in the entire system. When the user sends a message during active hours, the topic session MUST update the state.

```python
def register_response(state: dict):
    # 1. Reset escalation timing
    state["escalation"]["last_response_at"] = now_iso()
    
    # 2. Advance state machine
    if state["state"] == "INITIAL":
        state["state"] = "ENGAGED"
    
    # 3. Set last_acted_level:
    #    -1 = "I have something to say" (fires L0 next cycle)
    #     0 = "Nothing to say, just monitoring"
    if state["state"] in ("INITIAL", "ENGAGED"):
        state["escalation"]["last_acted_level"] = -1
    elif state["state"] == "HAS_PLAN":
        has_due = check_for_due_transitions(state)
        state["escalation"]["last_acted_level"] = -1 if has_due else 0
    else:
        state["escalation"]["last_acted_level"] = 0
    
    # 4. Skip past-due transitions if we were escalating
    if old_level > 0:
        mark_overdue_transitions_as_skipped(state)
```

**This function must be called on EVERY user message during active hours.** If it's not called, the system thinks the user is silent and escalates. This was the #1 bug across all versions.

---

## HAS_PLAN Suppression

When the user has a plan and no transitions are due, the system should go silent. Don't nag.

```python
def should_suppress(state: dict) -> bool:
    if state["state"] != "HAS_PLAN":
        return False
    
    plan = state.get("plan", {})
    current_time = datetime.now(TZ)
    
    # Check each unchecked transition
    for t in plan.get("transitions", []):
        if not t.get("checked"):
            check_at = datetime.fromisoformat(t["check_at"])
            if current_time >= check_at:
                return False  # transition is due, don't suppress
    
    # Check bedtime check
    if plan.get("bedtime_check_at"):
        bedtime_check = datetime.fromisoformat(plan["bedtime_check_at"])
        if current_time >= bedtime_check:
            return False  # bedtime check due, don't suppress
    
    return True  # nothing due, suppress escalation
```

**Both the CLI context command AND the enforcement script must check this.** If they get out of sync, one might suppress while the other escalates.

---

## Timezone Handling

Use `zoneinfo.ZoneInfo` (Python 3.9+) for all timezone work. Never use naive datetimes.

```python
from zoneinfo import ZoneInfo

# Load from config
TZ = ZoneInfo(config["user"]["timezone"])

def now():
    return datetime.now(TZ)

def now_iso():
    return now().isoformat()
```

**Rules:**
- All timestamps stored in state.json must include timezone offset
- When parsing ISO strings, always handle both tz-aware and tz-naive (add TZ if missing)
- The user's timezone from config is the single source of truth
- Never use UTC internally — everything is in the user's local time for readability
- Display times in 12-hour format with AM/PM for user-facing messages
