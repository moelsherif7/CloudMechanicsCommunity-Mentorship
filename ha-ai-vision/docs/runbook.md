# Runbook

Operate, troubleshoot, and validate the Home Assistant AI Vision pilot.

---

## Quick reference

| Thing | Value |
|---|---|
| Core script | `script.ai_vision_analyze` |
| Provider swap point | `input_text.llmvision_provider` |
| Cooldown helper | `input_datetime.front_door_last_ai` (≥5 min) |
| Camera | `camera.front_door` |
| Motion | `binary_sensor.front_door_motion` |
| Notify | `notify.mobile_app_mohamed` |
| Timeline | `calendar.llm_vision_timeline` *(verify exact id)* |
| Snapshot file | `/config/www/snapshots/front_door_latest.jpg` → `/local/snapshots/front_door_latest.jpg` |

---

## 1. Health checks (do these first)

1. **Integration loaded?** Settings → Devices & Services → **LLM Vision** present
   and not showing an error.
2. **Provider helper set?** Developer Tools → States →
   `input_text.llmvision_provider` holds a valid provider id.
3. **Secret present?** `secrets.yaml` contains the key referenced by your
   provider (e.g. `llmvision_gemini_api_key`). HA logs warn on missing secrets.
4. **Snapshot folder exists & is writable?** `/config/www/snapshots/`. Create it
   if missing (the `www` folder maps to `/local/`).
5. **Manual run:** Developer Tools → Actions → `script.ai_vision_analyze` with
   test fields (see §4). Confirm a notification arrives.

---

## 2. Troubleshooting

### No notification at all
- Confirm the automation **triggered**: Settings → Automations → open the
  automation → **Traces**. If no trace, the motion sensor isn't firing
  (`from: off → to: on`). Check `binary_sensor.front_door_motion` in States.
- Confirm not blocked by **cooldown**: check `input_datetime.front_door_last_ai`.
  If it was set within the last 5 minutes, the script `stop`s by design. Set it
  to an old time to test immediately.
- Confirm the **notify service** name. `notify.mobile_app_mohamed` must match
  your device exactly (Developer Tools → Actions → search `notify.`).

### I get the *fallback* "AI unavailable" message every time
This means `llmvision.image_analyzer` failed or returned empty. Check, in order:
- **Logs:** Settings → System → Logs, filter `llmvision` / `ai_vision`.
- **API key / quota:** invalid key, expired billing, or rate limit.
- **Service shape:** field names (`provider`, `image_file`, `message`,
  `response_variable`) can differ by version. Open Developer Tools → Actions →
  `llmvision.image_analyzer` and compare to `scripts/llmvision_helpers.yaml`.
- **Snapshot missing:** if `camera.snapshot` failed, there's no image to send.
  Verify the camera entity is available and the path is writable.

### Snapshot is black / stale
- Some cameras need a moment after waking. Increase any pre-snapshot delay, or
  use a stream-based snapshot. Confirm `camera.front_door` shows a live image on
  the dashboard.

### Timeline is empty / no calendar entity
- Ensure **Timeline** (and **Memory**) were enabled when configuring LLM Vision.
- Verify the calendar entity id; update it in `automations/05_daily_summary.yaml`
  and `dashboards/ai_vision_timeline.yaml` if it differs.

### Over-alerting (too many pushes)
- Raise `cooldown_minutes` for that camera.
- Tighten the prompt's `action` rules (e.g. only notify on `confidence: high`).
- Add known people/pets to **Memory** so they stop reading as "unknown".

### YAML won't load
- Run **Developer Tools → YAML → Check Configuration** before reloading.
- Keep 2-space indentation; ensure each automation file is a list item (`- id:`).

---

## 3. Cost monitoring

The cooldown is your spend cap. To keep an eye on usage:

1. **Count AI calls.** Each successful analysis stamps
   `input_datetime.front_door_last_ai`. For real accounting, add a
   `counter.ai_vision_calls` and `counter.increment` it inside
   `script.ai_vision_analyze` (right after the analysis step). Then graph it:
   ```yaml
   # configuration.yaml
   counter:
     ai_vision_calls:
       name: AI Vision calls
       icon: mdi:counter
   ```
2. **Set a daily budget alarm.** Add an automation: if `counter.ai_vision_calls`
   exceeds N per day, send a warning push. Reset the counter at midnight.
3. **Watch the provider console.** Gemini/OpenAI/Anthropic dashboards show
   token usage and spend; set billing alerts there too.
4. **Reduce cost levers** (see `architecture.md` §4): raise cooldown, lower
   snapshot resolution (`target_width`), keep prompts/outputs short, or move to
   **Ollama** (local, no per-call cost).

**Sanity math:** worst case ≈ `(60 / cooldown_minutes)` calls/hour/camera. At
5 min that's ≤12/hour ≈ ≤288/day if motion never stops — realistic days are far
lower.

---

## 4. Fallback test (validate graceful degradation)

Goal: prove that an AI failure still yields a motion notification + snapshot.

**A. Confirm the happy path**
1. Set `input_datetime.front_door_last_ai` to an old time (clears cooldown).
2. Developer Tools → Actions → `script.ai_vision_analyze`:
   ```yaml
   use_case: package
   camera_entity: camera.front_door
   notify_target: mobile_app_mohamed
   cooldown_helper: input_datetime.front_door_last_ai
   prompt: "Describe the doorstep. Then one JSON line: {\"category\":\"package\",\"present\":false,\"confidence\":\"low\",\"action\":\"notify\"}"
   ```
3. Expect a push with the snapshot and the AI summary.

**B. Force the failure path**
1. Temporarily break the provider: set `input_text.llmvision_provider` to a
   bogus value (e.g. `does_not_exist`), **or** revoke/blank the API key in
   `secrets.yaml` and reload.
2. Re-run the manual action from step A (clear cooldown again first).
3. **Expected:** you receive the **fallback** notification
   (*"Motion detected … AI analysis was unavailable. Snapshot attached."*) and a
   `warning` entry from logger `ai_vision` in the logs.
4. Restore the real provider id / key and reload.

**C. Confirm the cooldown gate**
1. Run a successful analysis (stamps the helper).
2. Immediately re-run within 5 minutes.
3. **Expected:** the script `stop`s with a "Cooldown active" message and **no**
   second AI call / notification.

If A, B, and C all behave as described, the pilot's reliability guarantees hold.

---

## 5. Routine maintenance

- **Prune snapshots:** the `snapshots/` folder grows over time and may contain
  personal data. Add a scheduled `shell_command` or `folder_watcher` cleanup, or
  delete files older than N days.
- **Prune Timeline:** keep retention to ~30 days (see provider config) to bound
  storage and limit how long footage-derived data is kept (relevant for
  UAE PDPL data-minimization — see `architecture.md` §5).
- **Rotate keys:** if a key leaks, revoke in the provider console and update
  `secrets.yaml`. Keys are never committed (`.gitignore`).
- **After HA/integration updates:** re-run the **fallback test** (§4) since
  service field names can change.
