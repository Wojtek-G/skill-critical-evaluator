# 🛡️ skill-critical-evaluator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A quality gate subagent skill for Claude Cowork/Code. It runs as an independent reviewer on an Opus-class model, evaluating task plans before execution and completed work before delivery — catching issues the implementing agent might miss.

Built for Claude, but the skill format can be adapted to any agentic harness that supports skill/instruction files.

## ✨ What it does

Runs at two gates during any multi-step task:

1. **Before execution** — reviews the plan for correctness, completeness, and hidden assumptions
2. **Before delivery** — reviews the finished work for mistakes, edge cases, and whether it actually solves the problem

Returns a structured **APPROVED** or **REJECTED** verdict with severity-ranked issues. On rejection, the implementing agent fixes and resubmits. The loop repeats until approved (5-iteration failsafe before escalating to the user).

## 🚀 Install

A skill is just a folder with a `SKILL.md` inside it. This repo ships the skill as `critical-evaluator.skill` — that file *is* the SKILL.md content.

**Claude Code / Cowork** — easiest path is to ask Claude to do it:

```
Clone https://github.com/Wojtek-G/skill-critical-evaluator and install
critical-evaluator.skill as a skill in my ~/.claude/skills directory.
```

Or do it manually:

```bash
git clone https://github.com/Wojtek-G/skill-critical-evaluator.git
mkdir -p ~/.claude/skills/critical-evaluator
cp skill-critical-evaluator/critical-evaluator.skill \
   ~/.claude/skills/critical-evaluator/SKILL.md
```

Use `.claude/skills/` inside a project instead of `~/.claude/skills/` to scope it to that project. Start a new session and Claude picks it up automatically.

**Claude.ai (paid plans)** — upload the skill in **Settings → Capabilities → Skills**.

## 🎯 Usage

No command needed — Claude loads it automatically at the plan and completion gates of a multi-step task. You can also trigger a review explicitly with `/critical-evaluator` or by asking Claude to "sanity-check this" / "make sure this is right."

## 🔧 Adapting to other agentic harnesses

The skill is plain text with structured instructions. To use it with a different agent framework:

- Copy the contents into your system prompt, agent instruction file, or tool definition
- Adjust the model references (`Opus`) to whatever reviewer model your harness supports
- Adapt the subagent invocation pattern to match how your harness spawns independent agents

## 📄 License

MIT
