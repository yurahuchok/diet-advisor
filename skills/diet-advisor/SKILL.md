---
name: diet-advisor
description: Use when the user runs any /diet-* command (diet-init, diet-fix, diet-start, diet-add, diet-remove, diet-summary, diet-mb, diet-ask) or asks to track calories, protein, fat, or carbs, log a meal or dish, or calculate daily intake targets.
---

# Diet Advisor

Tracks daily food intake against personalized calorie and macro targets calculated from the user's stats and goal.

## Memory

All Diet Advisor data lives in its own store at `~/.claude/diet-advisor/` — never in Claude's general memory, CLAUDE.md, or the project:

```
~/.claude/diet-advisor/
  profile.json          # stats, maintenance numbers, goal, daily targets
  days/YYYY-MM-DD.json  # one file per tracked day
```

- Get today's date with `date +%F`. Commands operate on today's file; if the user supplies a date (YYYY-MM-DD), use that day instead.
- Run `mkdir -p ~/.claude/diet-advisor/days` before the first write.
- If `profile.json` is missing and the command is not diet-init, stop and tell the user to run `/diet-init` first.
- Day totals are always computed by summing entries at read time — never stored.

`profile.json`:

```json
{
  "updated": "2026-08-17",
  "sex": "male", "age": 30, "weight_kg": 80.0, "height_cm": 180,
  "gym_sessions_per_week": 4, "activity_multiplier": 1.55,
  "bmr": 1780,
  "maintenance": {"calories": 2760, "protein_g": 144, "fat_g": 77, "carbs_g": 373},
  "goal": "fat_loss",
  "targets": {"calories": 2210, "protein_g": 176, "fat_g": 61, "carbs_g": 239}
}
```

`days/2026-08-17.json`:

```json
{
  "date": "2026-08-17",
  "entries": [
    {"id": 1, "dish": "Oatmeal with banana", "portion": "80 g oats, 1 medium banana",
     "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}
  ]
}
```

## Calculations

Round calories to the nearest 10 and grams to whole numbers (.5 rounds up); each step uses the already-rounded outputs of previous steps. Always present numbers as **kcal / protein g / fat g / carbs g**.

1. **BMR** (Mifflin-St Jeor): `10·weight_kg + 6.25·height_cm − 5·age + 5` for male, `− 161` instead of `+ 5` for female. If the user prefers not to say, average both results.
2. **Activity multiplier** from gym sessions/week: `0 → 1.2 · 1–2 → 1.375 · 3–4 → 1.55 · 5–6 → 1.725 · 7+ → 1.9`
3. **Maintenance calories** = BMR × activity multiplier.
4. **Goal targets**:

| Goal | Calories | Protein |
|---|---|---|
| fat_loss | maintenance × 0.80 | 2.2 g/kg |
| muscle_gain | maintenance × 1.10 | 2.0 g/kg |
| recomp (fat loss + muscle gain) | maintenance × 0.90 | 2.2 g/kg |
| maintenance | maintenance × 1.00 | 1.8 g/kg |

5. **Fat** = 25% of target calories ÷ 9, rounded to whole grams. **Carbs** = (target calories − 4·protein − 9·fat) ÷ 4, using the already-rounded protein and fat grams, then rounded to whole grams. Maintenance macros use the same fat/carb split with 1.8 g/kg protein.

Accept imperial units and convert for storage (1 lb = 0.4536 kg, 1 in = 2.54 cm). Sanity-check inputs (age 13–100, weight 35–250 kg, height 130–230 cm, gym 0–14/week) and ask the user to confirm anything outside these ranges.

## Estimating a dish

Use general nutrition knowledge. If the portion is unstated or ambiguous (e.g., rice by dry vs cooked weight), assume the most common interpretation, state the assumption ("assuming ~200 g cooked rice"), and invite correction. Report the dish's kcal/P/F/C before logging it.

## Status report

End diet-add, diet-remove, diet-summary, and diet-mb with:

1. A table: for each of calories, protein, fat, carbs → consumed, target, remaining. Flag negative remaining as over target.
2. One short rebalancing suggestion: which macros lag or exceed, plus 2–3 concrete foods that close the gap within the remaining calories (e.g., protein lagging with calories to spare → chicken breast, greek yogurt, egg whites; carbs lagging → rice, fruit, oats; fat lagging → nuts, olive oil, avocado, salmon).

## Commands

### diet-init
Ask one question at a time (use AskUserQuestion when available): biological sex (needed for the BMR formula), age, weight, height, gym sessions/week. Then compute BMR → maintenance calories → maintenance macros, save to `profile.json`, show the results, and ask if anything needs fixing. Then ask the goal (fat loss / muscle gain / fat loss + muscle gain / strict maintenance), compute goal targets, save, show them, and again ask if anything needs fixing. If `profile.json` already exists, warn before overwriting.

### diet-fix
Show the current profile values, apply the requested change(s). If any stat or the goal changed, recalculate BMR, maintenance, and targets, save, and show before → after so the user sees what the recalculation changed.

### diet-start
Create today's day file with empty entries. If it already exists with entries, ask before resetting it. If a dish is included with the command, immediately run the diet-add workflow on it; otherwise tell the user to add the first dish with `/diet-add`.

### diet-add
Ensure today's file exists (create it silently if not). Estimate the dish, append an entry with the next id, save, then show the dish's numbers followed by the status report.

### diet-remove
Show today's entries with ids if the target is ambiguous; remove the requested entry; save; status report. If the day has no entries, say so.

### diet-summary
Show a table of today's entries (id, dish, kcal/P/F/C), then the status report.

### diet-mb ("maybe")
Estimate the dish and show its numbers but do NOT save anything. Show a hypothetical status report ("If you eat this: …") — what the totals would be and what would remain. End by noting the dish was not logged and can be added with `/diet-add`.

### diet-ask
Answer the diet/nutrition question from general knowledge, using the profile and today's log as context. Never write to memory silently: if the answer implies changing stored data (stats, goal, targets, entries), propose the exact change and ask first. On approval, update, recalculate anything downstream, and show the result.

## Common mistakes

- Writing diet data anywhere except `~/.claude/diet-advisor/` — it must stay separate from general memory.
- Storing computed totals in the day file — always sum entries at read time.
- Changing weight or goal without recalculating maintenance and targets.
- Logging a dish during diet-mb — mb never writes.
- Guessing a portion silently — always state the assumed portion.
