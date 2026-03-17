# CLAUDE.md — Formula Agent Rules

## Project Overview
Formula is a volleyball analytics platform featuring PIR (Player Impact Rating) and Winning Formula — proprietary metrics that provide coaches with actionable statistical insights.

## Agent Behavior Rules
- NEVER stop to ask the user questions. Make your best judgment.
- NEVER change scoring algorithms or PIR calculations without explicit instruction.
- ALWAYS read existing code before writing anything new.
- ALWAYS match existing patterns and code style.
- ALWAYS use TypeScript.
- If blocked, log the issue to ~/logs/blocked.md and continue.
- When complete, write a summary to ~/logs/task-summary.md.

## Git Workflow
- Create feature branches for each task
- Do not merge into main
