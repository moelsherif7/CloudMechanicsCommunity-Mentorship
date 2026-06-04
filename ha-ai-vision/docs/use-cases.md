# Use Cases

The pilot ships three core security use cases plus two extras. This document is
the **backlog** of future AI Vision use cases — each is built the same way: a
prompt in `prompts/`, an automation that calls `script.ai_vision_analyze`, and
(optionally) a dedicated camera + cooldown helper. No core changes required.

Legend: ✅ shipped · 🔭 backlog · 💡 idea

---

## Shipped (pilot)

| # | Use case | Camera | Notify trigger | Files |
|---|---|---|---|---|
| ✅ | **Package detection** | front door | package present, med+ conf | `prompts/package.md`, `automations/01_package_detection.yaml` |
| ✅ | **Unknown person** | front door | unknown person, med+ conf | `prompts/person.md`, `automations/02_unknown_person.yaml` |
| ✅ | **Known pet vs. stray** | front door | unknown/stray animal | `prompts/animal.md`, `automations/03_pet_motion.yaml` |
| ✅ | **Room tidiness** | indoor | score ≤ 2 | `prompts/tidiness.md`, `automations/04_room_tidiness.yaml` |
| ✅ | **Daily evening summary** | n/a | 20:00 digest | `prompts/anomaly.md` (Role 2), `automations/05_daily_summary.yaml` |

---

## Backlog — Security & access

- 🔭 **Vehicle / driveway detection** — known vs. unknown car in the driveway;
  optionally read make/color (avoid storing plates for privacy/PDPL).
- 🔭 **Door left open / ajar** — periodic anomaly check flags an open door or
  gate when no one is around (anomaly prompt already supports this).
- 🔭 **Loitering detection** — same person present across multiple snapshots in a
  short window → escalate. Needs lightweight cross-snapshot memory.
- 🔭 **Tampering / obstruction** — camera covered, sprayed, or moved.
- 🔭 **Gate/garage status** — confirm garage door closed at night.
- 💡 **Weapon / threat flag** — high-stakes; require very high confidence and
  human confirmation before any automated action.

## Backlog — Deliveries & logistics

- 🔭 **Package removed / picked up** — pair with #01 to log when a delivered
  package is taken inside (or stolen). Compare consecutive Timeline events.
- 🔭 **Mail vs. parcel classification** — envelope/letter vs. box.
- 🔭 **Delivery courier identification** — which carrier (uniform/logo), logged
  to Timeline for "who delivered today?".
- 💡 **Mis-delivery alert** — package addressed to a neighbor left at your door.

## Backlog — Household & pets

- 🔭 **Pet at the door wanting in/out** — known pet waiting → notify to let in.
- 🔭 **Pet behavior / distress** — unusual posture or activity.
- 🔭 **Food/water bowl empty** — indoor camera checks bowls at set times.
- 💡 **Litter box / mess detection** — flag accidents for cleanup.

## Backlog — Safety & wellbeing

- 🔭 **Fall detection (common areas)** — person on the floor unexpectedly →
  high-priority alert. Sensitive; tune for low false positives.
- 🔭 **Stove/appliance left on** — visible indicator lights / open flame.
- 🔭 **Water leak / flooding** — standing water on floor (kitchen, bathroom).
- 🔭 **Smoke / fire visible cues** — complement, not replace, smoke detectors.
- 🔭 **Child near hazard** — pool gate, stairs, kitchen (privacy-sensitive).

## Backlog — Home & lifestyle

- 🔭 **Room tidiness trends** — extend #04 with weekly scoring and charts.
- 🔭 **Plant health** — leaf yellowing/wilting from a plant-shelf camera.
- 🔭 **Whiteboard / fridge note capture** — OCR notes into a to-do list.
- 🔭 **Laundry / dishwasher done** — read appliance status lights.
- 💡 **Fridge inventory** — rough "what's running low" from a fridge-cam.

## Backlog — Insights & reporting

- 🔭 **Weekly summary** — extend the daily digest to a Sunday weekly recap.
- 🔭 **Visitor log** — count and roughly categorize daily visitors.
- 🔭 **Heatmap of activity** — busiest hours at the front door.
- 💡 **Natural-language Q&A presets** — saved questions on the dashboard
  ("packages this week?", "any unknown persons overnight?").

---

## How to add a new use case (checklist)

1. **Write the prompt** — copy an existing `prompts/*.md`; keep the structured
   contract: ≤2 sentences + one JSON tag with `category`, `confidence`, `action`.
2. **Pick camera + motion source** — reuse front door, or add a new camera and a
   dedicated `input_datetime.<camera>_last_ai` cooldown helper.
3. **Add an automation** — copy an existing `automations/*.yaml`, give it a
   unique `id` and descriptive `alias`, set `use_case`, `camera_entity`,
   `prompt`, `notify_target`, and `cooldown_helper`.
4. **Tune cooldown** — security = 5 min; ambient checks (tidiness, plants) can be
   15–60 min to save cost.
5. **Test** — run the fallback + cooldown tests in `runbook.md` §4.
6. **Document** — move the item to "Shipped" above and update the README table.

---

## Prioritization guidance

| Priority | Pick use cases that… |
|---|---|
| **High** | run on cameras you already have, have clear notify rules, low false-positive risk (packages, unknown person, doors). |
| **Medium** | need a new camera or extra memory/state (loitering, package removed, tidiness trends). |
| **Careful** | are safety-critical or privacy-sensitive (fall detection, child-near-hazard, weapons) — require high confidence and, ideally, human confirmation before action. Mind UAE PDPL when footage of identifiable people is processed in the cloud (see `architecture.md` §5). |
