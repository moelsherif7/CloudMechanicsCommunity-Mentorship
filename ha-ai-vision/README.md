# Home Assistant AI Vision

> Event-driven, multimodal AI analysis of camera snapshots in Home Assistant.
> When motion fires, a snapshot is sent to an LLM, which returns a short
> structured description. Everything is logged to a **Timeline** you can query
> in natural language — e.g. *"did a package arrive today?"*

This project is **automation-first** and **provider-agnostic**. The pilot ships
with Google **Gemini 2.0 Flash**, but you can swap to OpenAI, Anthropic, or a
local **Ollama** model by changing a single config value.

---

## What it does

```
motion ──▶ cooldown gate ──▶ snapshot ──▶ LLM Vision ──▶ structured result
                                                │             │
                                                │             ├─▶ Timeline (queryable history)
                                                │             ├─▶ Memory (context for follow-ups)
                                                │             └─▶ mobile notification
                                                └─▶ on failure: plain motion alert + snapshot
```

### Pilot scope (built and working)

- **One camera:** front door (`camera.front_door`)
- **Three use cases:** package detection, unknown person, known pet
- **Timeline + Memory** enabled
- **Daily 20:00 evening summary** pushed to your phone

Plus two extras included in this repo: a **room tidiness** check and the
**daily summary** automation. A full backlog of future use cases lives in
[`docs/use-cases.md`](docs/use-cases.md).

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Home Assistant OS | Tested against recent HA OS releases |
| [HACS](https://hacs.xyz/) | Required to install the integration below |
| [LLM Vision](https://github.com/valentinfrlch/ha-llmvision) | `valentinfrlch/ha-llmvision`, installed via HACS |
| A camera entity | Pilot uses `camera.front_door` |
| A motion source | Pilot uses `binary_sensor.front_door_motion` (mmWave/PIR/camera-derived) |
| Mobile app | [HA Companion app](https://companion.home-assistant.io/) for `notify.mobile_app_*` |
| Provider API key | Gemini by default; or OpenAI / Anthropic / Ollama |

> **Placeholders used throughout:** `camera.front_door`,
> `binary_sensor.front_door_motion`, `notify.mobile_app_mohamed`.
> Replace these with your real entity IDs after reviewing.

---

## Install

1. **Install the integration**
   - HACS → Integrations → search **LLM Vision** → install → restart HA.
   - Settings → Devices & Services → **Add Integration** → *LLM Vision*.
   - Add your **provider** (Gemini) and enable **Timeline** and **Memory** when
     prompted. See [`config/llmvision_provider.example.yaml`](config/llmvision_provider.example.yaml)
     for the exact options and how they map to the UI.

2. **Add secrets** (never committed — see `.gitignore`)
   In your HA `secrets.yaml`:
   ```yaml
   llmvision_gemini_api_key: "AIza...your-key..."
   # Optional alternates:
   # llmvision_openai_api_key: "sk-..."
   # llmvision_anthropic_api_key: "sk-ant-..."
   ```

3. **Copy the project files into your HA config**
   - `scripts/llmvision_helpers.yaml` → merge into your `scripts.yaml`
     (or `script: !include_dir_merge_named scripts/`).
   - `automations/*.yaml` → merge into your `automations.yaml`
     (or `automation: !include_dir_merge_list automations/`).
   - `dashboards/ai_vision_timeline.yaml` → add as a new Lovelace view (raw
     config editor) or include as a dashboard.
   - `prompts/*.md` are the source of truth for the prompt text embedded in the
     automations — edit them there, then keep the automation copies in sync.

4. **Set your provider id**
   Each automation/script references `input_text.llmvision_provider` (the
   configured LLM Vision provider entry id). Create that helper or hard-code
   your provider id. See the swap section below.

5. **Reload & test**
   - Developer Tools → YAML → **Reload Scripts** and **Reload Automations**.
   - Run the **fallback test** in [`docs/runbook.md`](docs/runbook.md) to confirm
     both the AI path and the graceful-fallback path work.

---

## Swap the provider (one change)

The provider is referenced indirectly so you only change it in **one place**.

1. Configure the new provider in the LLM Vision integration UI (adds a new
   provider entry id).
2. Update the single helper `input_text.llmvision_provider` (Settings →
   Devices & Services → Helpers) to the new provider id — **done**. Every
   automation reads from this helper.

| Provider | `provider` value style | Key in `secrets.yaml` | Notes |
|---|---|---|---|
| Google Gemini 2.0 Flash *(default)* | LLM Vision provider id | `llmvision_gemini_api_key` | Fast, cheap, strong vision |
| OpenAI (GPT-4o / 4o-mini) | LLM Vision provider id | `llmvision_openai_api_key` | Swap model in provider config |
| Anthropic (Claude) | LLM Vision provider id | `llmvision_anthropic_api_key` | Strong reasoning |
| Ollama (local) | LLM Vision provider id | *(none — local)* | Data stays on-prem; needs a vision model e.g. `llava`/`llama3.2-vision` |

> Because prompts return a **provider-neutral structured contract**
> (≤2 sentences + a JSON tag), switching providers does not require touching
> the automations or the dashboard.

---

## Design rules (enforced in this repo)

- **Event-driven, no polling** — automations trigger on motion only.
- **Per-camera cooldown** — minimum **5 minutes** between AI calls (cost + spam control).
- **Graceful fallback** — if the LLM call fails, you still get a plain motion
  notification with the snapshot attached.
- **Secrets stay out of git** — `secrets.yaml`, `.storage`, `*.key` are ignored.
- **Short structured output** — every prompt returns ≤2 sentences plus a JSON
  tag like `{"category":"package","confidence":"high"}`.
- **Clean YAML** — 2-space indent, unique `id` + descriptive `alias` per automation.

---

## Repo layout

```
ha-ai-vision/
├── README.md                         ← you are here
├── docs/
│   ├── architecture.md               ← flow, components, cost, UAE data residency
│   ├── use-cases.md                  ← full backlog
│   └── runbook.md                    ← troubleshooting, cost monitoring, fallback test
├── automations/
│   ├── 01_package_detection.yaml
│   ├── 02_unknown_person.yaml
│   ├── 03_pet_motion.yaml
│   ├── 04_room_tidiness.yaml
│   └── 05_daily_summary.yaml
├── prompts/
│   ├── package.md
│   ├── person.md
│   ├── animal.md
│   ├── tidiness.md
│   └── anomaly.md
├── scripts/
│   └── llmvision_helpers.yaml        ← reusable snapshot + analyze + log script
├── dashboards/
│   └── ai_vision_timeline.yaml
├── config/
│   └── llmvision_provider.example.yaml
└── .gitignore
```

---

## Safety & privacy

Camera snapshots can contain people, faces, and license plates. This repo
ignores captured media and `.storage` by default. If you use a cloud provider,
images leave your network — read the **data residency** note in
[`docs/architecture.md`](docs/architecture.md) (includes a UAE-specific note).
For strict on-prem requirements, use the **Ollama** provider.

---

## License

Provided as community lab content for the Cloud Mechanics Community Mentorship.
Use at your own risk; review before deploying to your home network.
