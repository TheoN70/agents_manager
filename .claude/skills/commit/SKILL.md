---
name: commit
effort: low
description: |
  MANDATORY - REPLACES manual git commit.

  This skill is the ONLY way to create git commits. NEVER use manual `git commit` commands.

  NON-NEGOTIABLE RULES:
  - ALL commits MUST go through this skill
  - NEVER run `git commit -m "..."` directly - always use /commit
  - User requests for "quick commit" or "commit rapidement" still use this skill
  - This ensures consistent commit message format across the repository

  AUTOMATIC TRIGGERS:
  - User says "commit", "save my work"
  - User wants to save changes to git
  - After completing a task that modified files (commit only, NEVER auto-push)

  WORKFLOW: Always invoke this Skill → Never manual git commit
allowed-tools: Bash(git :*)
---

Indique le type de commit entre crochets ([FIX],[REFACTO],[FEATURE],[DOCS],[TEST],[CHORE]), puis un message court pertinant.
Utilise git diff pour voir les modifications.
Description en français en minuscule. Sans point.

Si rien n'est commité, lance un git add sur les fichiers ajoutés/modifiés.

Push sur la branche actuelle.

