# Diet Advisor

An [Agent Skill](https://agentskills.io) that tracks daily food intake against personalized calorie and macro targets. Works in any assistant that reads the SKILL.md format — Claude, ChatGPT, Gemini CLI, and others.

All data lives in a single portable `DIET ADVISOR DATA` block. On assistants with persistent memory the skill keeps the block there; anywhere else it prints the updated block after every change and you carry it between sessions. The block is also the migration format — paste it into another assistant running this skill and continue where you left off.

## Layout

```
skills/diet-advisor/SKILL.md     # The skill: storage modes, formulas, workflows
.claude-plugin/plugin.json       # Claude plugin manifest
.claude-plugin/marketplace.json  # Claude marketplace serving the plugin
```

## Install

**claude.ai / Claude Desktop** (Pro/Max/Team/Enterprise): Customize → Plugins → "+" → Add marketplace from GitHub → this repo's URL → install `diet-advisor`. On the Free plan (no plugin support), zip the `skills/diet-advisor` folder and upload it via Settings → Skills instead (requires code execution enabled).

**ChatGPT**: Plugins → Skills → Create → Upload, choosing the `skills/diet-advisor` folder. Availability depends on plan (GA on managed plans, beta elsewhere), and skills don't sync between devices — install on each one.

**Claude Code, Gemini CLI, Codex CLI, and other Agent Skills tools**: copy the `skills/diet-advisor` folder into the tool's skills directory (e.g. `~/.claude/skills/` for Claude Code; see your tool's docs).

**Anything else**: paste the body of `SKILL.md` as custom instructions. The data block keeps your data portable regardless.

## Usage

The skill is conversational — just ask:

- Set up or fix your diet profile (stats → maintenance intake → goal → daily targets)
- Start a new diet day
- Log a dish, remove a mistakenly logged one, or correct its values later
- Preview a dish's impact without logging it ("maybe")
- Show today's intake, remaining macros, and suggestions
- Look back at the 3 most recently logged days ("what did I have for breakfast yesterday?"; older days are dropped by design)
- Ask nutrition questions in the context of your data
- Wipe all Diet Advisor data ("delete everything") — one confirmation, then gone

Safety: the skill declines to set up tracking for minors, during pregnancy/breastfeeding, or for underweight cutting, and warns on very low calorie targets — those cases are referred to a professional.

## Method

Targets come from fixed published sources, cited to the user during setup:

- BMR via Mifflin-St Jeor (rated most accurate for healthy adults by the Academy of Nutrition and Dietetics), TDEE via standard activity factors keyed to gym frequency
- Goal adjustments: fat loss −20%, muscle gain +10%, recomp −10%, maintenance ±0 (per the ISSN position stand on diets and body composition)
- Protein anchored to bodyweight, 1.8–2.2 g/kg by goal (per the ISSN position stand on protein and exercise); fat 25% of calories (within the IOM 20–35% AMDR); carbs the remainder
