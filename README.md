# Diet Advisor

A Claude skill for claude.ai and Claude Desktop chat that tracks daily food intake against personalized calorie and macro targets, storing data in a dedicated Diet Advisor section of Claude memory (claude.ai has no persistent filesystem).

## Layout

```
.claude-plugin/plugin.json       # diet-advisor plugin manifest
.claude-plugin/marketplace.json  # Marketplace serving the plugin
skills/diet-advisor/SKILL.md     # Core skill: memory layout, formulas, workflows
```

## Install

Requires a plan with plugin support (Pro/Max/Team/Enterprise): Customize → Plugins → "+" → Add marketplace from GitHub → this repo's URL → install `diet-advisor`.

On the Free plan (no plugin support), zip the `skills/diet-advisor` folder and upload it manually via Settings → Skills instead (requires code execution enabled).

## Usage

The skill is conversational — just ask:

- Set up or fix your diet profile (stats → maintenance intake → goal → daily targets)
- Start a new diet day
- Log a dish, or remove a mistakenly logged one
- Preview a dish's impact without logging it ("maybe")
- Show today's intake, remaining macros, and suggestions
- Ask nutrition questions in the context of your data

## Method

- BMR via Mifflin-St Jeor, TDEE via gym-frequency activity multipliers
- Goal adjustments: fat loss −20%, muscle gain +10%, recomp −10%, maintenance ±0
- Protein anchored to bodyweight (1.8–2.2 g/kg by goal), fat 25% of calories, carbs the remainder
