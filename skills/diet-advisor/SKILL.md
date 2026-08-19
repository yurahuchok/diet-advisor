---
name: diet-advisor
description: Use when the user wants to track calories or macros, log or preview a meal, set up or fix a diet profile, review today's intake, or ask nutrition questions in the context of their diet data.
---

# Diet Advisor

Tracks daily food intake against personalized calorie and macro targets calculated from the user's stats and goal.

This skill is platform-agnostic: it runs in any assistant that reads Agent Skills. All data lives in a single portable data block; the Storage section defines where that block is kept depending on what this assistant can do. The Safety limits section overrides everything else in this skill.

## Storage

All Diet Advisor data lives in one **DIET ADVISOR DATA** block, structured exactly like this — keep the serialization compact, one line per entry and per dish:

```
DIET ADVISOR DATA
mode: memory
profile: {"updated": "2026-08-17",
  "sex": "male", "age": 30, "weight_kg": 80.0, "height_cm": 180,
  "gym_sessions_per_week": 4, "activity_multiplier": 1.55,
  "bmr": 1780,
  "maintenance": {"calories": 2760, "protein_g": 144, "fat_g": 77, "carbs_g": 373},
  "goal": "fat_loss",
  "targets": {"calories": 2210, "protein_g": 176, "fat_g": 61, "carbs_g": 239}}
today: {"date": "2026-08-17", "entries": [
  {"id": 1, "dish": "Oatmeal with banana", "portion": "80 g oats, 1 medium banana", "time": "breakfast", "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}]}
history:
  2026-08-16: {"target": {"calories": 2210, "protein_g": 176, "fat_g": 61, "carbs_g": 239}, "entries": [
    {"id": 1, "dish": "Chicken rice bowl", "portion": "300 g", "time": "13:30", "calories": 620, "protein_g": 45, "fat_g": 14, "carbs_g": 78}]}
dishes: [
  {"dish": "Oatmeal with banana", "portion": "80 g oats, 1 medium banana", "last_eaten": "2026-08-17", "calories": 420, "protein_g": 12, "fat_g": 7, "carbs_g": 82}]
```

### Modes

Where the block lives depends on this assistant's capabilities. Record the choice in the block's `mode` line so later sessions inherit it; re-decide only if the platform's capabilities have changed.

- **Memory mode** (`mode: memory`) — use only if you have a persistent memory feature that survives across conversations AND stores text verbatim; a paraphrasing or summarizing auto-memory does not qualify. Keep the block in a single clearly labeled Diet Advisor memory section and update it there. If the user references prior tracking but no memory section exists, say that the stored data appears to be lost and offer to restore from a pasted block before offering fresh setup. Offer a printed backup of the block when the user asks or at a natural end of session.
- **Data-block mode** (`mode: data-block`) — if you have no qualifying memory feature, or you are not certain writes survive the end of the conversation, treat memory as unavailable. On the first diet request, use the current block (below) or ask the user to paste their latest one. At the end of any reply that changed data, print the full updated block ONCE in a code fence — covering all changes made in that reply — and remind the user to save it (pinned note, text file — anywhere they can paste it from next time).

### The current block

- The current block is whichever block appears LAST in the conversation — the newest assistant-printed or user-pasted one. Exception: if a newly pasted block is older than one already present (compare `profile.updated` and `today.date`), warn the user and ask which is authoritative instead of silently adopting the stale one. If those dates are equal or mixed but the content differs, treat the blocks as divergent — the divergent-block rule below takes precedence over last-wins. In memory mode, the memory section's copy counts as a block already present: compare a pasted block against it under these same rules and write the adopted winner back to memory.
- Merging two divergent blocks is unsupported. If the user offers a second, different block (e.g., from another device), ask them to pick one authoritative block; never combine entries from both.
- One person per block. If a pasted block clearly belongs to someone else, do not adopt, merge, or store it; suggest they run this skill in their own session (the block is the migration format there).
- Never silently adopt a truncated or malformed block. Report which sections parsed and which did not, and ask for a repaste — or explicit confirmation to continue from the salvaged part.
- Legacy blocks from older versions of this skill are converted on first use, and the user is told so: a totals-only history line (`2026-08-16: 2180 kcal / 170 P / 58 F / 240 C (target …)`) becomes a history day with its `target` snapshot (if present) and one synthetic entry `{"id": 1, "dish": "(day total)", "calories": 2180, "protein_g": 170, "fat_g": 58, "carbs_g": 240}`; absent sections initialize deterministically: `mode` is re-decided per Modes, `dishes` starts empty, entry `time` stays absent.

### Rules (both modes)

- Read the current block before answering any diet request. If there is no profile, stop and offer to set one up first. The block is authoritative over any other remembered diet facts (e.g., a stale "user weighs 85 kg" memory elsewhere).
- Write the updated block by the end of any reply that changed it (setup, fix, log, remove, correction, new day) — to memory, or printed once at the end of the reply. The block is the only store; anything not written there is lost when the conversation ends. Never deliberately store diet data outside this block, and never mix unrelated memories into it.
- Today's date = the current date at message time when the platform provides it; otherwise the date stated in conversation context. If the stored `today` date is in the past, first move that day into `history` — its entries plus a `target` snapshot of the targets it was tracked against, taken before applying any other change requested in the same message — then start a fresh `today`. This rollover is automatic and never asks for confirmation. If the stored `today` date is in the future, flag the inconsistency and ask the user before proceeding.
- Every day keeps full entries with ids, so single dishes can be removed and past days reviewed; history days also snapshot that day's `target`. Keep the 3 most recently logged past days in `history`, dropping the oldest — this window is by design.
- A day holds at most 30 entries (memory management, also by design). If a day already has 30, don't log another dish into it — explain the limit and offer combine-or-replace (mechanics in the Log workflow).
- Entry ids: next id = highest id present in that day + 1, and within a conversation the counter never goes backward (a removal doesn't lower it). An id still present in the block is never reused, so logging order stays readable after removals.
- Each entry records a `time` when one is known: a clock time ("08:30") or a meal label ("breakfast", "lunch", "dinner", "late snack"). Map time-of-day words to the nearest meal label (morning → breakfast, midday → lunch, evening → dinner, night → late snack). Determine it invisibly, in the background, from the user's wording or the conversation context — never ask the user for a time, and never announce the inference. The message's own clock time counts as context only when the wording implies the dish was just eaten. For an ambiguous clock time ("at 8"), pick the reading consistent with context and logging order; if still ambiguous, store the nearest meal label instead. When nothing is available, omit the field; the entry's numeric `id` already records logging order, which is enough to reason about meals later.
- `dishes` is a reuse library with one entry per dish. Dish identity is semantic — translations and synonyms are the same dish — and the library keeps its existing canonical name and usual portion on refresh; a one-off portion variant never overwrites the saved usual portion unless the user says the new portion is their new usual. Every logged dish is added or refreshed and stamped with `last_eaten` (the date it was eaten). Keep at most 50 — also by design — evicting the dish with the oldest `last_eaten` when the cap is hit; on ties, evict the one listed first. A refresh updates a dish in place; new dishes append to the end (list position is the tie-break), and logged entries use the library's canonical dish name. If the user corrects a dish's values, update its library entry too. Removing a logged entry does not change the library or its stamps.
- Day totals — today's or a history day's — are always computed by summing entries at read time, never stored.

## Safety limits

These override every other rule in this skill:

- **Minors**: if the user is under 18, do not set up tracking or compute targets — even if they confirm their age. Explain that calorie tracking for minors needs professional supervision and refer them to a pediatrician or registered dietitian.
- **Pregnancy, breastfeeding, medical conditions**: out of scope — the pinned equations are not validated there, and deficits can be contraindicated. Decline setup (or a goal change into a deficit) and refer to a professional. If such a condition surfaces mid-use, stop prescribing targets and say why.
- **Underweight**: compute BMI (weight_kg ÷ height_m²) at setup and whenever weight changes. BMI < 18.5 with goal fat_loss or recomp → decline the deficit, say why, and refer to a professional. BMI < 17 → decline setup or further targets entirely, for any goal, with the same referral. If a weight Fix trips these limits on an existing profile, replace deficit targets with maintenance targets (BMI < 18.5) or clear `targets` (BMI < 17), say why, and refer — never leave deficit targets computed from old stats.
- **Low calorie targets**: if any calorie target — computed by the method or requested by the user — is below the user's BMR, or below 1200 kcal (female or unspecified) / 1500 kcal (male), do not apply it silently: warn, recommend professional guidance, and apply only if the user explicitly confirms after the warning. Never apply a target below 1000 kcal — decline and refer instead.

## Calculations

The complete method is specified below and set in stone: use exactly these equations, constants, and reasoning. Never substitute alternatives you may know, even better-seeming ones. Where a source gives a range, the specific value this skill picks inside it is fixed too. Cite the sources briefly when presenting setup results (the Setup workflow shows how); Fix recalculations do not repeat the citation.

Rounding, fully pinned: round calorie values (including BMR) to the nearest 10 and gram values to whole numbers; ALL halves round up, in every rounding this skill does — grams ending in .5, calorie values ending in 5, and stored weight/height halves. Each step uses the already-rounded outputs of previous steps, and the rounded values are what the block stores. Store weight to 0.1 kg and height to the nearest cm, converting imperial first (1 lb = 0.4536 kg, 1 in = 2.54 cm); calculations use the stored values. Calorie factors: 4 kcal/g protein, 4 kcal/g carbs, 9 kcal/g fat. Always present numbers as **kcal / protein g / fat g / carbs g**.

1. **BMR** — Mifflin-St Jeor equation (Mifflin et al., 1990), rated most accurate for healthy adults by the Academy of Nutrition and Dietetics: it predicted within 10% of measured resting metabolic rate more often than any competing equation, with the narrowest error range.
   `BMR = 10·weight_kg + 6.25·height_cm − 5·age + 5` for male; same with `− 161` in place of `+ 5` for female. If the user prefers not to say, average the two RAW results before any rounding (equivalently: use `− 78` as the final constant), then round; store `"sex": "unspecified"` and keep using this averaging in every recalculation.

2. **Maintenance calories (TDEE)** = BMR × activity factor, on the standard 1.2–1.9 clinical scale: sedentary 1.2, light activity 1.375, moderate 1.55, hard 1.725, very hard 1.9. This skill's fixed mapping from gym sessions/week: `0 → 1.2 · 1–2 → 1.375 · 3–4 → 1.55 · 5–6 → 1.725 · 7+ → 1.9`.

3. **Goal targets** — calorie adjustments per the ISSN Position Stand on Diets and Body Composition (Aragon et al., 2017): moderate deficits (~20%, pacing loss at roughly 0.5–1% of bodyweight per week) retain lean mass better than aggressive cuts, and muscle gain warrants only a small surplus (~10%) because larger surpluses mostly add fat. Protein per the ISSN Position Stand on Protein and Exercise (Jäger et al., 2017): 1.4–2.0 g/kg/day for exercising individuals, pushed to the top of that range or above while in a deficit to protect lean mass.

| Goal | Calories | Protein | Reasoning |
|---|---|---|---|
| fat_loss | maintenance × 0.80 | 2.2 g/kg | moderate deficit; protein above 2.0 to protect lean mass while cutting |
| muscle_gain | maintenance × 1.10 | 2.0 g/kg | small surplus; protein at the top of the general range |
| recomp (fat loss + muscle gain) | maintenance × 0.90 | 2.2 g/kg | mild deficit, so deficit-level protein |
| maintenance | maintenance × 1.00 | 1.8 g/kg | upper-middle of the general range |

4. **Fat** = 25% of target calories ÷ 9, rounded to whole grams — 25% is the midpoint of the Institute of Medicine / Dietary Guidelines Acceptable Macronutrient Distribution Range of 20–35% of calories from fat. **Carbs** = (target calories − 4·protein − 9·fat) ÷ 4, using the already-rounded protein and fat grams, then rounded to whole grams — carbs take the remainder because neither source fixes them and they fuel training. Maintenance macros use the same fat/carb split with 1.8 g/kg protein.

5. **Feasibility guard** (skill convention): if carbs come out below 50 g, lower the protein allotment in 0.1 g/kg steps — never below 1.4 g/kg, the ISSN floor — to the largest value that brings carbs to at least 50 g, and tell the user protein was reduced to keep the plan feasible. If carbs still fall short at 1.4 g/kg, the pinned method does not fit this profile: say so and refer to a professional instead of presenting a broken plan. The guard applies to maintenance macros and goal targets alike, and to macros re-derived after a manual target override. Every computed target also passes the Low-calorie-targets safety check.

Sanity checks: age 18–100 (under 18 → Safety limits, do not proceed), weight 35–250 kg, height 130–230 cm, gym 0–14/week. Ask the user to confirm any value outside the numeric ranges except age below 18, which is never accepted.

## Estimating a dish

First check the `dishes` library:

- An unqualified mention of a saved dish's name with no conflicting portion counts as that saved dish — reuse its numbers.
- References like "same as yesterday's lunch" resolve to the history entry first, then use its library dish's current (possibly corrected) values; if the dish is no longer in the library, copy the history entry's numbers.
- If more than one saved dish plausibly matches, ask which one — dish disambiguation is allowed (only time questions are forbidden).
- A stated portion that differs from the saved one: scale only the changed component when the relationship is clear (e.g., 120 g oats vs the saved 80 g — the banana is unchanged); a single-component dish scales as a whole; anything unclear, estimate fresh. The library keeps the saved usual portion either way.
- Whenever saved numbers are reused or scaled, tell the user they come from their saved dishes and can be corrected if the dish has changed. Anything less than solid evidence: estimate fresh.

When estimating fresh, use general nutrition knowledge. If the portion is unstated or ambiguous (e.g., rice by dry vs cooked weight), assume the most common interpretation, state the assumption ("assuming ~200 g cooked rice"), and invite correction. Report the dish's kcal/P/F/C before logging it.

## Status report

End the log, remove, correction, summary, and preview workflows with:

1. A table: for each of calories, protein, fat, carbs → consumed, target, remaining. Flag negative remaining as over target. For a past day, use that day's `target` snapshot.
2. One short rebalancing suggestion: which macros lag or exceed, plus 2–3 concrete foods that close the gap within the remaining calories (e.g., protein lagging with calories to spare → chicken breast, greek yogurt, egg whites; carbs lagging → rice, fruit, oats; fat lagging → nuts, olive oil, avocado, salmon).

Presentation rules are session defaults: if the user asks for terser output (e.g., "just say logged"), honor it for the rest of the session. Don't store presentation preferences in the block. The data-block print is a storage rule, not presentation — terseness never suppresses it.

## Workflows

This skill defines no commands — match the user's request to a workflow below. When one message asks for several actions, execute in this order: day rollover (snapshot first) → profile fix → removes and corrections → logs → questions; then save and, in data-block mode, print the block once at the end.

### Setup — "set up my diet", "calculate my targets" (no profile yet)
Ask one question at a time: biological sex (needed for the BMR formula; "prefer not to say" is accepted), age, weight, height, gym sessions/week. Apply Safety limits (age, BMI) before computing. Then compute BMR → maintenance calories → maintenance macros, save the block (set `profile.updated` to today), show the results, and ask if anything needs fixing. Then ask the goal (fat loss / muscle gain / fat loss + muscle gain / strict maintenance), compute goal targets (feasibility guard included), save, show them, and again ask if anything needs fixing. When presenting maintenance and goal targets, include a one- or two-line citation of the fixed sources from Calculations (e.g., "BMR via Mifflin-St Jeor, the Academy of Nutrition and Dietetics' accuracy pick; protein and calorie adjustments per the ISSN position stands; fat within the IOM 20–35% range") — not a bibliography. If a profile already exists, warn before overwriting.

### Fix — the user corrects a stat, goal, or target, or asks to recompute
Show the current profile values, apply the requested change(s), and set `profile.updated` to today. If any stat or the goal changed, recalculate BMR, maintenance, and targets (Safety limits and feasibility guard included), save, and show before → after so the user sees what the recalculation changed. A recompute request with unchanged stats ("recalculate my targets") recalculates from the stored stats the same way — no re-interview.
Manual target override (the user demands a specific number): run the Low-calorie-targets safety check first. If it passes (or the user explicitly confirms after the warning), apply it, re-derive fat (25% of the new calories) and carbs (remainder formula) while keeping protein per the goal, run the feasibility guard on the result, mark the profile with `"manual_targets": true`, note the deviation from the pinned method, and show before → after. On any later recalculation, ask whether to keep the manual targets or recompute them before clobbering.

### New day — "start a new day", "starting fresh today"
If the stored `today` is a past date, the storage rules roll it into history automatically — no confirmation. If the stored `today` is already the current date and has entries, warn that resetting discards today's entries (a same-dated day never moves to history) and proceed only on explicit confirmation; in a multi-action message, ask this first and defer the remaining actions until the user answers. Then create today with empty entries. If a dish is mentioned in the same request, immediately run the log workflow on it; otherwise invite the user to log their first dish.

### Log a dish — "I ate…", "log…", "add…"
Ensure today exists in the block (create it silently if not). Estimate the dish (library first), append an entry with the next id and a silently inferred `time` if one is available, update the `dishes` library per the storage rules, save, then show the dish's numbers followed by the status report.
- **Cap**: if the day already has 30 entries, stop and offer combine-or-replace. Combine = add the new dish's kcal/P/F/C into the entry the user picks, append the new dish's name to that entry's `dish` and its portion text to that entry's `portion`, keep the entry's id and `time`; the combined-in dish still gets its library add/refresh. Replace = remove the chosen entry, then log normally.
- **Several dishes in one message**: one entry per dish occurrence — the same dish at two stated times is two entries — with `time` inferred per item; dishes the user presents as one combined dish stay one entry. The cap applies per item: log what fits, list what was refused and why. One save, one status report, one block print for the whole batch.
- **Past day** ("log yesterday's dinner: …"): append to that stored history day using that day's next id; show that day's status report (vs its `target` snapshot). Logging to a past day that is not stored is declined, citing the 3-day window. A dish logged just after midnight belongs to the previous day when the wording says so ("before bed", "with dinner").
- **Just previewed**: logging a dish previewed earlier in this conversation logs the previewed numbers verbatim — the library write and `last_eaten` stamp happen now.

### Remove a dish — "remove…", "I didn't eat…"
Works on today or a stored history day. If the target entry is ambiguous, show that day's entries with ids, ask which to remove, and wait for the answer. Remove the entry, save, and show that day's status report (vs its own targets). Removal never changes the library. If the day has no entries, say so.

### Correct a dish — "that oatmeal was actually 600 kcal", "the portion was 200 g"
Find the entry (today or a stored history day; disambiguate by id and wait if unclear). Update the entry's values, save, and show that day's status report. A value correction updates the dish's library entry too; a portion-only correction rescales the logged entry but leaves the library's usual portion unless the user says it is their new usual. If the dish is no longer in the library, correct only the entry — don't re-add it. Corrections apply where the entry lives — including history days, since the user is explicitly asking.

### Summary — "how am I doing today?", "show my intake"
Show a table of today's entries (id, dish, time if known, kcal/P/F/C), then the status report.

### History — "what did I eat yesterday?", "what did I have for breakfast on Monday?", "how was this week?"
Answer from stored days — and the same meal-matching applies to questions about today. Show the day's dishes (with `time` values) and computed totals vs that day's `target` snapshot (today uses current targets). For meal-specific questions, match entries by `time` — meal labels directly, clock times by common mealtime ranges. For entries with no `time`, reason from logging order (ascending `id`): earliest entries lean breakfast, midday ones lunch, last ones dinner; present order-based answers as a stated assumption ("going by logging order, this was likely your breakfast") and invite correction. Multi-day questions ("this week") are answered per-day from today plus the stored days. If a requested day is not stored, say so and show which days are available: absent days newer than the oldest stored day were never logged; older days are outside the retained window (dropped if they were ever logged). Note that Diet Advisor keeps only the 3 most recently logged past days by design.

### Preview ("maybe") — "what if I eat [dish]…", "should I eat [a specific dish]…", "check this dish"
Estimate the dish (library reuse allowed, but write nothing — no `last_eaten` refresh) and show its numbers. Show a hypothetical status report ("If you eat this: …") — what the totals would be and what would remain. End by noting the dish was not logged and offering to log it (which would use these exact numbers).

### Reset — "delete all my diet data", "forget everything", "wipe diet advisor"
Ask exactly one confirmation, stating the full scope: profile, today's log, history, and the dish library will be permanently deleted. No other questions, no partial-deletion counteroffers, no talking the user out of it; an explicit confirmation already in the same message ("delete everything, yes I'm sure") counts, and the propose-and-confirm rule is satisfied by this single confirmation. On confirmation:
- **Memory mode**: delete the entire Diet Advisor memory section — the whole block, including `mode` — plus any other Diet Advisor remnants visible in memory. Note honestly that platform-level auto-memories outside your control may persist and where the user can clear those.
- **Data-block mode**: nothing is stored on the assistant's side — say so, stop using and printing the block for the rest of the conversation, and remind the user to delete any saved block copies they carry (pinned notes, files).
Then confirm completion. The skill is back to the no-profile state: the next diet request gets the setup offer.

### Diet question — any other nutrition question
Answer using the profile and today's log as context. Questions about the skill's own numbers ("why is my protein so high?") are answered from the pinned Calculations reasoning and sources, not from conflicting general knowledge. Requests to see or clear the dish library are honored here — clearing requires explicit confirmation. Never update the block silently: if the answer implies changing stored data (stats, goal, targets, entries, dishes), propose the exact change and ask first. On approval, apply stat, goal, or target changes via the Fix workflow (including its manual-override rules); apply entry or dish changes via the Correct workflow; then show the result.

## Common mistakes

- Answering from a stale in-conversation copy of the data instead of the current block, or changing data without immediately writing it — the block is the only store.
- In data-block mode, ending a data-changing reply without one full compact block print — or printing it multiple times in one reply.
- Deliberately storing diet data outside the DIET ADVISOR DATA block, letting unrelated memories leak into it, or letting a stray remembered fact override the block.
- Silently adopting a stale, truncated, or someone else's block — each has an explicit rule in The current block.
- Storing computed totals for any day — always sum entries at read time.
- Keeping more than 3 past days, letting a day exceed 30 entries, or letting `dishes` exceed 50 — all caps are by design, and refusals they cause should say so.
- Reusing a saved dish on a loose match, or reusing one silently — reuse follows the Estimating rules and must be mentioned as correctable.
- Logging (not previewing) a dish without refreshing its `last_eaten` — stale stamps evict the wrong dish at the 50 cap.
- Changing weight or goal without recalculating maintenance and targets — or recalculating over `"manual_targets": true` without asking.
- Deviating from the pinned equations, constants, rounding, or reasoning in Calculations — the method is set in stone — or presenting setup numbers without the brief citation.
- Skipping a Safety limit because the user insists — the limits override user confirmation except where they explicitly allow a confirmed override.
- Wiping data without the single confirmation — or stalling a confirmed Reset with extra questions, partial offers, or persuasion.
- Logging a dish during a preview — preview never writes.
- Guessing a portion silently — always state the assumed portion.
- Asking the user about time, or announcing time inferences while logging — `time` is determined invisibly from wording and context.
- Treating missing `time` as unanswerable for meal questions — logging order supports a best guess, stated as an assumption.
