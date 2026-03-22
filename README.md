# LLM Implementation Plans

Executable documentation for AI coding agents.

## What is this?

This repository contains markdown files that describe **complete implementation plans** — step-by-step instructions that an LLM-powered coding agent (Claude Code, Cursor, Copilot, etc.) can follow to set up, deploy, and configure real software systems on your machine.

Think of them as **installation scripts written in natural language**. Instead of running a bash script, you feed a markdown file to your AI agent and let it do the work — reading the instructions, making decisions based on your environment, and executing each step.

## Why markdown instead of a script?

Shell scripts are brittle. They assume a specific OS, package manager, directory layout, and don't adapt well when something unexpected happens. A markdown plan, interpreted by a capable LLM agent, can:

- **Adapt to your environment** — detect your OS, available disk space, existing services, and adjust accordingly
- **Make judgment calls** — choose between variants, skip optional steps, or stop and ask when something is ambiguous
- **Recover from errors** — read error output, diagnose the problem, and try an alternative approach
- **Explain what it's doing** — you can watch each step and intervene if needed

## How to use

1. Browse the plans in this repo and pick one
2. Open your LLM coding agent (e.g. Claude Code)
3. Point it at the plan:
   ```
   Follow the implementation plan in LocalWikipedia/IMPLEMENTATION_PLAN.md
   ```
4. The agent reads the plan and executes each step, asking for input when decisions are needed (e.g. which variant to install, what directory to use)

## Plans

| Plan | Description |
|------|-------------|
| [LocalWikipedia](LocalWikipedia/IMPLEMENTATION_PLAN.md) | Deploy a self-hosted Wikipedia mirror using Kiwix in Docker, with automated updates |

## Contributing

To add a new plan, create a directory with an `IMPLEMENTATION_PLAN.md` that includes:

- A clear goal statement
- Any relevant method comparisons or trade-off analysis
- Step-by-step implementation instructions with code blocks
- Configuration options and decision points
- Verification steps so the agent can confirm each phase worked

Plans should be written so that both a human and an LLM agent can follow them.
