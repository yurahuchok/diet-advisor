---
name: diet-advisor
description: Use when the user wants to track calories or macros, log or preview a meal, set up or fix a diet profile, review today's intake, or ask nutrition questions in the context of their diet data.
---

# Diet Advisor

Tracks daily food intake against personalized calorie and macro targets calculated from the user's stats and goal.

This skill is platform-agnostic: it runs in any assistant that reads Agent Skills. All data lives in a single portable data block; the Storage section defines where that block is kept depending on what this assistant can do.

## Storage

All Diet Advisor data lives in one **DIET ADVISOR DATA** block, structured exactly like this:

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
   "time": "breakfast",
   "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}]}
history:
  2026-08-16: {"target": {"calories": 2210, "protein_g": 176, "fat_g": 61, "carbs_g": 239},
    "entries": [
      {"id": 1, "dish": "Chicken rice bowl", "portion": "300 g", "time": "13:30",
       "calories": 620, "protein_g": 45, "fat_g": 14, "carbs_g": 78}]}
dishes: [
  {"dish": "Oatmeal with banana", "portion": "80 g oats, 1 medium banana",
   "last_eaten": "2026-08-17",
   "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}]
```

Where the block lives depends on this assistant's capabilities — pick one mode and stay in it:

- **Memory mode** — if you have a persistent memory feature that survives across conversations, keep the block in a single clearly labeled Diet Advisor memory section and update it there.
- **Data-block mode** — if you have no such feature, or you are not certain writes survive the end of the conversation, treat memory as unavailable. On the first diet request, look for the block earlier in the conversation or ask the user to paste their latest one. After every change, print the full updated block in a code fence and remind the user to save it (pinned note, text file — anywhere they can paste it from next time).

The block is also the migration format: pasted into any other assistant running this skill, it carries all data over.

Rules (both modes):

- Read the current block before answering any diet request. If there is no profile, stop and offer to set one up first.
- After every change (setup, fix, log, remove, new day), write the updated block immediately — to memory, or printed for the user. The block is the only store; anything not written there is lost when the conversation ends.
- Use today's date from the conversation context. If the stored `today` date is in the past, first move that day into `history` — its entries plus a `target` snapshot of the targets it was tracked against — then start a fresh `today`.
- Every day keeps full entries with ids, so single dishes can be removed and past days reviewed; history days also snapshot that day's `target`. Keep at most 3 past days in `history`, dropping the oldest — this 3-day window is by design.
- A day holds at most 30 entries (memory management, also by design). If `today` already has 30, don't log another dish — explain the limit and offer to combine it into an existing entry or replace one.
- Each entry records a `time` when one is known: a clock time ("08:30") or a meal label ("breakfast", "late snack"). Determine it invisibly, in the background, from the user's wording or the conversation context — never ask the user for a time, and never announce the inference. When nothing is available, omit the field; the entry's numeric `id` already records logging order, which is enough to reason about meals later.
- `dishes` is a reuse library: every logged dish is added to it, or refreshed if already present, and stamped with `last_eaten` (today's date). Keep at most 50 — also by design — evicting the dish with the oldest `last_eaten` when the cap is hit; the stamp exists precisely so the eviction choice is unambiguous. If the user corrects a dish's values, update its library entry too.
- Day totals — today's or a history day's — are always computed by summing entries at read time, never stored.
- Never store diet data outside this block, and never mix unrelated memories into it.

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

First check the `dishes` library. Reuse a saved dish's numbers only on solid evidence that it is exactly that dish — same dish and portion, or the user saying so ("my usual oatmeal", "same as yesterday's lunch"). When reusing, tell the user the numbers come from their saved dishes and can be corrected if the dish has changed. Anything less than solid evidence: estimate fresh.

When estimating fresh, use general nutrition knowledge. If the portion is unstated or ambiguous (e.g., rice by dry vs cooked weight), assume the most common interpretation, state the assumption ("assuming ~200 g cooked rice"), and invite correction. Report the dish's kcal/P/F/C before logging it.

## Status report

End the log, remove, summary, and preview workflows with:

1. A table: for each of calories, protein, fat, carbs → consumed, target, remaining. Flag negative remaining as over target.
2. One short rebalancing suggestion: which macros lag or exceed, plus 2–3 concrete foods that close the gap within the remaining calories (e.g., protein lagging with calories to spare → chicken breast, greek yogurt, egg whites; carbs lagging → rice, fruit, oats; fat lagging → nuts, olive oil, avocado, salmon).

## Workflows

This skill defines no commands — match the user's request to a workflow below.

### Setup — "set up my diet", "calculate my targets"
Ask one question at a time: biological sex (needed for the BMR formula), age, weight, height, gym sessions/week. Then compute BMR → maintenance calories → maintenance macros, save the block, show the results, and ask if anything needs fixing. Then ask the goal (fat loss / muscle gain / fat loss + muscle gain / strict maintenance), compute goal targets, save, show them, and again ask if anything needs fixing. If a profile already exists, warn before overwriting.

### Fix — the user corrects a stat, goal, or target
Show the current profile values, apply the requested change(s). If any stat or the goal changed, recalculate BMR, maintenance, and targets, save, and show before → after so the user sees what the recalculation changed.

### New day — "start a new day", "starting fresh today"
Move any previous `today` into history (per the storage rules), then create today with empty entries. If today already exists with entries, ask before resetting it. If a dish is mentioned in the same request, immediately run the log workflow on it; otherwise invite the user to log their first dish.

### Log a dish — "I ate…", "log…", "add…"
Ensure today exists in the block (create it silently if not). If today already has 30 entries, stop: explain the 30-dish daily limit and offer to combine the dish into an existing entry or replace one. Otherwise estimate the dish, append an entry with the next id and a silently inferred `time` if one is available, update the `dishes` library per the storage rules, save, then show the dish's numbers followed by the status report.

### Remove a dish — "remove…", "I didn't eat…"
Show today's entries with ids if the target is ambiguous; remove the requested entry; save; status report. If the day has no entries, say so.

### Summary — "how am I doing today?", "show my intake"
Show a table of today's entries (id, dish, time if known, kcal/P/F/C), then the status report.

### History — "what did I eat yesterday?", "what did I have for breakfast on Monday?"
Answer from the stored history days: that day's dishes (with their `time` values) and its computed totals vs its `target` snapshot. For meal-specific questions, match entries by `time` — meal labels directly, clock times by common mealtime ranges. For entries with no `time`, reason from logging order (ascending `id`): earliest entries lean breakfast, midday ones lunch, last ones dinner. Present order-based answers as a stated assumption ("going by logging order, this was likely your breakfast") and invite correction. If the requested day is older than the stored window, say that Diet Advisor keeps only the last 3 days by design and show which days are available.

### Preview ("maybe") — "what if I eat…", "should I eat…", "check this dish"
Estimate the dish and show its numbers but do NOT save anything. Show a hypothetical status report ("If you eat this: …") — what the totals would be and what would remain. End by noting the dish was not logged and offering to log it.

### Diet question — any other nutrition question
Answer from general knowledge, using the profile and today's log as context. Never update the block silently: if the answer implies changing stored data (stats, goal, targets, entries), propose the exact change and ask first. On approval, update, recalculate anything downstream, and show the result.

## Common mistakes

- Answering from a stale in-conversation copy of the data instead of the current block, or changing data without immediately writing it — the block is the only store.
- In data-block mode, updating data without printing the refreshed block — the user leaves the conversation with stale data.
- Storing diet data outside the DIET ADVISOR DATA block, or letting unrelated memories leak into it.
- Storing computed totals for any day — always sum entries at read time.
- Keeping more than 3 past days, letting a day exceed 30 entries, or letting `dishes` exceed 50 — all caps are by design, and refusals they cause should say so.
- Reusing a saved dish on a loose match, or reusing one silently — reuse needs solid evidence, and must be mentioned as correctable.
- Logging or reusing a dish without refreshing its `last_eaten` — stale stamps evict the wrong dish at the 50 cap.
- Changing weight or goal without recalculating maintenance and targets.
- Logging a dish during a preview — preview never writes.
- Guessing a portion silently — always state the assumed portion.
- Asking the user about time, or announcing time inferences while logging — `time` is determined invisibly from wording and context.
- Treating missing `time` as unanswerable for meal questions — logging order supports a best guess, stated as an assumption.
