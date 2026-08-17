# Diet Advisor

A Claude Code skill that tracks daily food intake against personalized calorie and macro targets, with its own memory separate from Claude's general memory.

## Layout

```
.claude-plugin/plugin.json       # diet-advisor plugin manifest (Claude Code)
.claude-plugin/marketplace.json  # Marketplace serving both plugins in this repo
skills/diet-advisor/SKILL.md     # Core skill: file-based storage, formulas, command workflows
commands/diet-*.md               # 8 slash-command wrappers
diet-advisor-web/                # diet-advisor-web plugin (claude.ai / Claude Desktop chat)
  .claude-plugin/plugin.json
  skills/diet-advisor/SKILL.md   # Same tracker, data stored in Claude memory
```

## Install

This repo is a plugin marketplace serving two variants of the same tracker — install the one matching where you use Claude.

**Claude Code** (`diet-advisor` — file-based storage, slash commands):

```
/plugin marketplace add /path/to/diet-advisor   # or owner/repo once on GitHub
/plugin install diet-advisor@diet-advisor
```

Commands are namespaced: `/diet-advisor:diet-add`, `/diet-advisor:diet-summary`, etc.

**claude.ai / Claude Desktop chat** (`diet-advisor-web` — Claude-memory storage, conversational; Pro/Max/Team/Enterprise): Customize → Plugins → "+" → Add marketplace from GitHub → this repo's URL → install `diet-advisor-web`. On the Free plan (no plugin support), zip the `diet-advisor-web/skills/diet-advisor` folder and upload it manually via Settings → Skills instead (requires code execution enabled).

Diet data stays out of Claude's general memory in the Claude Code variant (`~/.claude/diet-advisor/`: `profile.json` + `days/YYYY-MM-DD.json`); the web variant stores it in a dedicated Diet Advisor section of Claude memory, since claude.ai has no persistent filesystem.

## Commands

| Command | Purpose |
|---|---|
| `/diet-init` | One-by-one setup questions → maintenance intake → goal → daily targets |
| `/diet-fix` | Fix any stored value; recalculates downstream numbers |
| `/diet-start` | Start a new diet day (optionally logging the first dish) |
| `/diet-add <dish>` | Log a dish; shows day totals, remaining, and rebalancing suggestions |
| `/diet-remove <dish>` | Remove a mistakenly logged dish; recalculates |
| `/diet-summary` | Today's intake table, remaining macros, suggestions |
| `/diet-mb <dish>` | "Maybe" — preview a dish's impact without logging it |
| `/diet-ask <question>` | Nutrition Q&A in the context of your data; asks before saving anything |

## Method

- BMR via Mifflin-St Jeor, TDEE via gym-frequency activity multipliers
- Goal adjustments: fat loss −20%, muscle gain +10%, recomp −10%, maintenance ±0
- Protein anchored to bodyweight (1.8–2.2 g/kg by goal), fat 25% of calories, carbs the remainder
