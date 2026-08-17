---
name: diet-advisor
description: Use when the user wants to track calories or macros, log or preview a meal, set up or fix a diet profile, review today's intake, or ask nutrition questions in the context of their diet data.
---

# Diet Advisor

Tracks daily food intake against personalized calorie and macro targets calculated from the user's stats and goal.

This is the claude.ai variant: there is no filesystem that persists between conversations, so all Diet Advisor data lives in Claude's memory, in one clearly labeled section.

## Memory

Keep all Diet Advisor data in a single **Diet Advisor** memory section, structured exactly like this:

```
DIET ADVISOR DATA
profile: {"updated": "2026-08-17",
  "sex": "male", "age": 30, "weight_kg": 80.0, "height_cm": 180,
  "gym_sessions_per_week": 4, "activity_multiplier": 1.55,
  "bmr": 1780,
  "maintenance": {"calories": 2760, "protein_g": 144, "fat_g": 77, "carbs_g": 373},
  "goal": "fat_loss",
  "targets": {"calories": 2210, "protein_g": 176, "fat_g": 61, "carbs_g": 239}}
today: {"date": "2026-08-17", "entries": [
  {"id": 1, "dish": "Oatmeal with banana", "portion": "80 g oats, 1 medium banana",
   "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}]}
history:
  2026-08-16: 2180 kcal / 170 P / 58 F / 240 C (target 2210/176/61/239)
```

Rules:

- Read this memory section before answering any diet request. If there is no profile, stop and offer to set one up first.
- After every change (setup, fix, log, remove, new day), update the memory section immediately. Memory is the only store — anything not written there is lost when the conversation ends.
- Use today's date from the conversation context. If the stored `today` date is in the past, first compress that day into a one-line history summary (date + total kcal/P/F/C vs target), then start a fresh `today`.
- `today` keeps full entries with ids so a single dish can be removed; `history` keeps totals only. Keep at most 14 history lines, dropping the oldest.
- Day totals for `today` are always computed by summing entries at read time — never stored.
- Never store diet data outside this section, and never mix general memories into it.

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

End the log, remove, summary, and preview workflows with:

1. A table: for each of calories, protein, fat, carbs → consumed, target, remaining. Flag negative remaining as over target.
2. One short rebalancing suggestion: which macros lag or exceed, plus 2–3 concrete foods that close the gap within the remaining calories (e.g., protein lagging with calories to spare → chicken breast, greek yogurt, egg whites; carbs lagging → rice, fruit, oats; fat lagging → nuts, olive oil, avocado, salmon).

## Workflows

There are no slash commands on claude.ai — match the user's request to a workflow below.

### Setup — "set up my diet", "calculate my targets"
Ask one question at a time: biological sex (needed for the BMR formula), age, weight, height, gym sessions/week. Then compute BMR → maintenance calories → maintenance macros, save to memory, show the results, and ask if anything needs fixing. Then ask the goal (fat loss / muscle gain / fat loss + muscle gain / strict maintenance), compute goal targets, save, show them, and again ask if anything needs fixing. If a profile already exists in memory, warn before overwriting.

### Fix — the user corrects a stat, goal, or target
Show the current profile values, apply the requested change(s). If any stat or the goal changed, recalculate BMR, maintenance, and targets, save, and show before → after so the user sees what the recalculation changed.

### New day — "start a new day", "starting fresh today"
Summarize any previous `today` into history, then create today with empty entries. If today already exists with entries, ask before resetting it. If a dish is mentioned in the same request, immediately run the log workflow on it; otherwise invite the user to log their first dish.

### Log a dish — "I ate…", "log…", "add…"
Ensure today exists in memory (create it silently if not). Estimate the dish, append an entry with the next id, save, then show the dish's numbers followed by the status report.

### Remove a dish — "remove…", "I didn't eat…"
Show today's entries with ids if the target is ambiguous; remove the requested entry; save; status report. If the day has no entries, say so.

### Summary — "how am I doing today?", "show my intake"
Show a table of today's entries (id, dish, kcal/P/F/C), then the status report.

### Preview ("maybe") — "what if I eat…", "should I eat…", "check this dish"
Estimate the dish and show its numbers but do NOT save anything. Show a hypothetical status report ("If you eat this: …") — what the totals would be and what would remain. End by noting the dish was not logged and offering to log it.

### Diet question — any other nutrition question
Answer from general knowledge, using the profile and today's log as context. Never write to memory silently: if the answer implies changing stored data (stats, goal, targets, entries), propose the exact change and ask first. On approval, update, recalculate anything downstream, and show the result.

## Common mistakes

- Answering from an earlier in-conversation copy of the data instead of the memory section, or changing data without immediately writing memory — memory is the only store.
- Writing diet data outside the Diet Advisor memory section, or letting general memories leak into it.
- Storing computed totals for today — always sum entries at read time (history lines are the one exception: totals only).
- Changing weight or goal without recalculating maintenance and targets.
- Logging a dish during a preview — preview never writes.
- Guessing a portion silently — always state the assumed portion.
