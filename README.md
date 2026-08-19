# Calorie Tracker

An [Agent Skill](https://agentskills.io) that logs dishes and does arithmetic against your daily calorie and macro targets. Works in any assistant that reads the SKILL.md format — Claude, ChatGPT, Gemini CLI, and others.

**Strictly a calorie tracker.** It counts numbers: intake logged, targets, remaining budget, and example foods whose standard nutrition values fit what's left. It is not a medical or health tool — it gives no health or dietary advice, refers all health questions to qualified professionals, and every reply ends with a disclaimer saying so.

All data lives in a single portable `CALORIE TRACKER DATA` block. On assistants with persistent memory the skill keeps the block there; anywhere else it prints the updated block after every change and you carry it between sessions. The block is also the migration format — paste it into another assistant running this skill and continue where you left off. Blocks from this skill's predecessor (Diet Advisor) are converted automatically.

## Layout

```
skills/calorie-tracker/SKILL.md  # The skill: storage modes, formulas, workflows
.claude-plugin/plugin.json       # Claude plugin manifest
.claude-plugin/marketplace.json  # Claude marketplace serving the plugin
.codex-plugin/plugin.json        # OpenAI (ChatGPT / Codex) plugin manifest
.agents/plugins/marketplace.json # Repo marketplace serving the plugin to ChatGPT / Codex
assets/icon.png, assets/logo.png # Square plugin icon/logo (required by the OpenAI directory)
```

## Install

**claude.ai / Claude Desktop** (Pro/Max/Team/Enterprise): Customize → Plugins → "+" → Add marketplace from GitHub → this repo's URL → install `calorie-tracker`. On the Free plan (no plugin support), zip the `skills/calorie-tracker` folder and upload it via Settings → Skills instead (requires code execution enabled).

**ChatGPT / Codex (as a plugin)**: clone this repo — its `.agents/plugins/marketplace.json` is a repo marketplace serving the plugin. In ChatGPT desktop, enable developer mode (Settings → Security and login → Developer mode), restart the app, and install `calorie-tracker` from the Plugins Directory under the local marketplace source. Or add the marketplace to `~/.agents/plugins/marketplace.json` to make it available everywhere (`source.path` resolves relative to the marketplace root).

**ChatGPT (skill upload only)**: Plugins → Skills → Create → Upload, choosing the `skills/calorie-tracker` folder. Availability depends on plan (GA on managed plans, beta elsewhere), and skills don't sync between devices — install on each one.

**Claude Code, Gemini CLI, Codex CLI, and other Agent Skills tools**: copy the `skills/calorie-tracker` folder into the tool's skills directory (e.g. `~/.claude/skills/` for Claude Code; see your tool's docs).

**Anything else**: paste the body of `SKILL.md` as custom instructions. The data block keeps your data portable regardless.

## Usage

The skill is conversational — just ask:

- Set up your targets (stats → fixed-formula maintenance estimate → pick an arithmetic preset → daily numbers)
- Start a new tracking day
- Log a dish, remove a mistakenly logged one, or correct its values later
- Preview a dish's impact without logging it ("maybe")
- Show today's intake and what's left of the day's numbers
- Look back at the 3 most recently logged days ("what did I have for breakfast yesterday?"; older days are dropped by design)
- Wipe all data ("delete everything") — one confirmation, then gone

Out of scope by design: health, medical, and dietary-advice questions are not answered — the skill refers them to a doctor or registered dietitian. It also declines to compute targets for minors and in other situations listed in the skill's Scope limits, with the same referral.

## Method

Target numbers are arithmetic outputs of fixed published formulas — the skill presents them as estimates, never as personal or medical guidance:

- BMR via the Mifflin-St Jeor equation; maintenance (TDEE) via the standard 1.2–1.9 activity multipliers keyed to gym frequency
- Four fixed target presets, chosen by the user: −20%, −10%, ±0, +10% of maintenance, each with a fixed protein constant (2.2 / 2.2 / 1.8 / 2.0 g per kg)
- Fat set at 25% of target calories; carbs take the remainder

*Not medical advice — Calorie Tracker only does the math. For any health-related questions, consult a medical professional.*
