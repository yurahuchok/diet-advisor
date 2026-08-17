# Diet Advisor

A Claude Code skill that tracks daily food intake against personalized calorie and macro targets, with its own memory separate from Claude's general memory.

## Layout

```
.claude-plugin/plugin.json      # Plugin manifest
.claude-plugin/marketplace.json # Makes this repo installable as its own marketplace
skills/diet-advisor/SKILL.md    # Core skill: memory schema, formulas, command workflows
commands/diet-*.md              # 8 slash-command wrappers
```

## Install

This repo is a Claude Code plugin and doubles as its own marketplace:

```
/plugin marketplace add /path/to/diet-advisor   # or a git URL / owner/repo once shared
/plugin install diet-advisor@diet-advisor
```

Commands are namespaced when installed as a plugin: `/diet-advisor:diet-add`, `/diet-advisor:diet-summary`, etc.

Diet data is stored in `~/.claude/diet-advisor/` (`profile.json` + `days/YYYY-MM-DD.json`), never in Claude's general memory.

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
