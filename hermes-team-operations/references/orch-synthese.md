# orch synthese — v4.2 Buffer Implementation

Added 06/07/2026, upgraded to buffer system same day.
Trigger: `orch synthese` in the vitrine.

## Flow (v4.2 — buffer + filtre anti-bruit)

1. `handle_orch_command()` returns `("Synthese demandee...", "synthese")`
2. Main loop enters the `synthese` action handler:
   - Gets `cycle_history[-1]` (last cycle responses) and `cycle_history[:-1]` (context)
   - Builds a structured prompt: question + all agent responses + previous cycles + synthesis request
   - Sends to Chris's private room via `send_to(PRIVATE_ROOMS["@chris:videocours.fr"], prompt)`
   - Sets `st["synth_waiting"] = True` + `st["synth_deadline"] = time.time() + 180`
3. **Phase capture (buffer)**: when Chris sends messages in his private room:
   - Noise patterns (gateway warnings, skill loading, retry msgs) → ignored, logged
   - Valid messages → appended to `synth_buffer[]`, `synth_last_msg` timestamp updated
4. **Phase finalisation**: main loop checks every ~3s (POLL_INT):
   - If 8s since last Chris message AND buffer not empty → join buffer messages, post in vitrine AND all private rooms as `🧠 Synthèse de CHRIS :`, clear state
   - If 8s since last Chris message AND buffer empty → log "buffer vide (que du bruit)", clear state
5. **Timeout** (3 min): checked in Timeouts section → posts `⏰ Synthese timeout`, clears buffer state

## Noise patterns filtered

```python
NOISE_PATTERNS = [
    'No home channel is set for Matrix',    # gateway warning
    'Reading skill',                        # skill loading
    'hermes-team-operations',               # specific skill load
    'Empty response from model',            # LLM retry notification
    'Model returned no content',            # LLM failure
    'No reply: the model returned empty',   # gateway empty response
]
```

## Code locations (orchestrator_v4.py)

- Command routing: `handle_orch_command()` → action `"synthese"` handler (~line 480)
- Buffer capture: ~line 540 (before `HORS CYCLE` check, after `agent/sender` check)
- Buffer finalization: ~line 607 (in main loop, before `time.sleep(POLL_INT)`)
- Timeout: ~line 582 (in Timeouts section, resets `synth_waiting`)
- NOISE_PATTERNS declaration: after imports, ~line 20

## State keys

| Key | Type | Purpose |
|---|---|---|
| `synth_waiting` | bool | True while waiting for Chris's synthesis |
| `synth_deadline` | float | Unix timestamp of 3-min deadline |
| `synth_buffer` | list[str] | Accumulates valid messages from Chris |
| `synth_last_msg` | float | Unix timestamp of last buffered Chris message |

## Dependencies

- Chris must be connected to Matrix and reachable in his private room `!aAHxSomr9iHrxsxZ:videocours.fr`
- `PRIVATE_ROOMS` mapping must include `"@chris:videocours.fr"`
- `NOISE_PATTERNS` must be defined (global constant)

## Bugs fixed

### Bug #1: Capture code commented out by cleanup script (06/07/2026)
**Symptom**: `orch synthese` → timeout every time, Chris ignored.
**Cause**: A Python cleanup script (`fix_orch_clean.py`) commented out the entire synth capture block because its indentation (16 spaces) differed from the surrounding code.
**Fix**: Uncommented the block and realigned indentation.
**Lesson**: After any bulk indentation-fixing, grep for `synth_waiting` to verify the capture block is active (no leading `#`).

### Bug #2: First-message capture swallows gateway noise (06/07/2026)
**Symptom**: `SYNTHESE recue de CHRIS: 193 chars` containing `"📬 No home channel is set for Matrix..."` — real synthesis sent 30s later was ignored as `HORS CYCLE`.
**Cause**: Chris's gateway sends 3 messages: noise → noise → real synthesis. The first valid message (not filtered by `is_system_msg()`) was captured, clearing `synth_waiting`, so the real synthesis arriving 30s later was treated as HORS CYCLE.
**Fix**: Implemented buffer + noise filtering + 8s silence finalization (this document).
