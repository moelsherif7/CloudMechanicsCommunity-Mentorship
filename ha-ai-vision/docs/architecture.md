# Architecture

How the Home Assistant AI Vision pilot is wired, what each component does,
what it costs, and where the data lives.

---

## 1. End-to-end flow

```
┌──────────────┐   motion    ┌───────────────────────┐
│ mmWave / PIR │────────────▶│ Automation trigger     │
│ camera motion│             │ (binary_sensor → on)   │
└──────────────┘             └──────────┬────────────┘
                                        │
                            ┌───────────▼────────────┐
                            │ Cooldown gate          │  ← per-camera, ≥5 min
                            │ input_datetime compare │     (skips if too soon)
                            └───────────┬────────────┘
                                        │ allowed
                            ┌───────────▼────────────┐
                            │ script.ai_vision_analyze│
                            │  1. camera.snapshot     │
                            │  2. llmvision.image_analyzer (Timeline+Memory)
                            │  3. parse JSON tag      │
                            │  4. notify              │
                            └───────────┬────────────┘
                          success │            │ failure
                    ┌─────────────▼──┐   ┌─────▼──────────────────┐
                    │ Timeline event │   │ Fallback: plain motion │
                    │ + Memory       │   │ notification + snapshot│
                    │ + mobile push  │   └────────────────────────┘
                    └────────────────┘
```

**Key properties**

- **Event-driven:** nothing polls the camera; analysis only happens on motion.
- **Cooldown:** a per-camera timestamp helper enforces a ≥5-minute gap between
  AI calls to control cost and notification spam.
- **Single choke point:** all use cases call one reusable script
  (`script.ai_vision_analyze`), so behavior, logging, and fallback are
  consistent everywhere.
- **Graceful degradation:** if the provider errors or times out, you still get
  a motion alert with the snapshot — you are never blind.

---

## 2. Components

| Component | Role | Where defined |
|---|---|---|
| **Motion sensor** | Event source (`binary_sensor.front_door_motion`) | HA device (Zigbee/Matter/mmWave) |
| **Camera** | Image source (`camera.front_door`) | HA camera integration |
| **Cooldown helper** | `input_datetime.front_door_last_ai` last-run timestamp | HA helper |
| **Provider selector** | `input_text.llmvision_provider` — the active provider id | HA helper |
| **LLM Vision integration** | Calls the model, manages **Timeline** + **Memory** | HACS: `valentinfrlch/ha-llmvision` |
| **Reusable script** | `script.ai_vision_analyze` — snapshot → analyze → log → notify | `scripts/llmvision_helpers.yaml` |
| **Automations** | One per use case; each calls the script with a prompt | `automations/*.yaml` |
| **Prompts** | Structured-output instructions per use case | `prompts/*.md` |
| **Dashboard** | Lovelace timeline + status card | `dashboards/ai_vision_timeline.yaml` |
| **Notifier** | `notify.mobile_app_mohamed` (HA Companion app) | HA mobile integration |

### Timeline vs. Memory

- **Timeline** — a chronological, queryable log of AI observations. This is what
  powers natural-language questions like *"did a package arrive today?"*.
- **Memory** — supplies context to the model (e.g. known people/pets, "what the
  porch normally looks like") so descriptions are more accurate and you get
  fewer false "unknown person" alerts.

---

## 3. The structured-output contract

Every prompt must return **≤2 sentences** of human-readable text followed by a
single-line JSON tag. Example:

```
A delivery person left a medium cardboard box by the front door. No one else is present.
{"category":"package","confidence":"high","action":"notify"}
```

Why this matters:

- **Provider-neutral:** the same contract works across Gemini, OpenAI,
  Anthropic, and Ollama, so swapping providers needs **no** automation changes.
- **Machine-parsable:** automations extract `category`/`confidence` to decide
  whether to escalate (push), log silently, or ignore.
- **Cheap:** short outputs mean fewer output tokens and faster responses.

See each file in [`../prompts/`](../prompts/) for the exact per-use-case spec.

---

## 4. Cost notes

Costs are dominated by **how often you call the model**, which the cooldown gate
directly controls. Vision requests bill for the input image (tokenized by
resolution) plus a small prompt and a short output.

**Levers that reduce cost**

1. **Cooldown (≥5 min/camera):** the single biggest lever. Bounds worst-case
   calls to ~12/hour/camera even under constant motion.
2. **Snapshot resolution:** smaller images = fewer input tokens. Use a modest
   snapshot size; you rarely need full sensor resolution for "is there a box".
3. **Short prompts + short outputs:** the structured contract keeps both small.
4. **Use a fast/cheap model:** Gemini 2.0 Flash is the default for this reason.

**Rough order-of-magnitude (illustrative, not a quote)**

| Scenario | AI calls/day | Relative cost |
|---|---|---|
| Quiet door, cooldown on | ~10–30 | cents/day on Flash-class models |
| Busy door, cooldown on | ~100–150 (capped by cooldown) | low single-digit $/month |
| **No cooldown (don't)** | thousands | unbounded |

> Always confirm current pricing with your provider. The point of the
> architecture is that **you can cap spend with the cooldown** and **switch to a
> cheaper or local model with one change**. See cost-monitoring steps in
> [`runbook.md`](runbook.md).

**Local option:** Ollama removes per-call cost entirely (you pay in hardware /
electricity and accept lower accuracy than frontier vision models).

---

## 5. Data residency & privacy

Camera snapshots may contain **people, faces, and license plates** — treat them
as personal data.

- **Cloud providers (Gemini / OpenAI / Anthropic):** snapshots are transmitted
  to the provider for inference. Images may transit and be processed in regions
  outside your country, subject to the provider's data-handling and retention
  terms. Review whether they use submitted data for training and whether you can
  opt out / use an enterprise tier with no-retention guarantees.
- **Local provider (Ollama):** images never leave your LAN. Choose this when you
  need strict on-prem processing.

### UAE data-residency note 🇦🇪

If you are operating in the **UAE**, personal data (which camera footage of
identifiable individuals is) may fall under the **UAE Federal Decree-Law No. 45
of 2021 on the Protection of Personal Data (PDPL)** and, for some sectors/free
zones (e.g. DIFC, ADGM), additional data-protection regimes. Practical guidance:

- Sending snapshots to a cloud LLM may constitute a **cross-border transfer** of
  personal data. Confirm the provider offers an adequate safeguard / acceptable
  region, and document your lawful basis (e.g. consent of household members).
- For the strictest residency requirement, run **Ollama locally** so footage
  **never leaves the UAE / never leaves your home network**.
- Minimize retention: this repo's `.gitignore` already excludes captured media;
  prune snapshots and Timeline history on a schedule (see `runbook.md`).
- Inform members of the household that AI vision is active.

> This is operational guidance, **not legal advice**. Confirm obligations with a
> qualified advisor for your specific situation.

---

## 6. Failure modes & handling

| Failure | Detection | Handling |
|---|---|---|
| Provider timeout / 5xx | script `continue_on_error` + error branch | Plain motion notification + snapshot |
| Invalid / missing JSON tag | parse step yields no `category` | Treat as low-confidence; still notify with raw text |
| Snapshot fails (camera offline) | `camera.snapshot` errors | Notify "motion but camera unavailable" |
| Motion storm | cooldown gate | Subsequent triggers skipped until window elapses |
| Wrong/over-eager alerts | Memory + confidence threshold | Only push on `confidence: high` for sensitive categories |

---

## 7. Extending

Add a new use case by:

1. Writing a new `prompts/<name>.md` following the structured contract.
2. Adding an automation that calls `script.ai_vision_analyze` with that prompt,
   its own camera, and (optionally) its own cooldown helper.

No changes to the core script, provider config, or dashboard are required.
The backlog in [`use-cases.md`](use-cases.md) lists ready-to-build ideas.
