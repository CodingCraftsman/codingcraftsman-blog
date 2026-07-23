---
title: 'When Your AI Agent Destroys Its Own Blueprint: Lessons from a Git Worktree
  Incident'
date: '2026-07-23T02:28:48.463112-04:00'
tags:
- lumis
- engineering
description: "A real incident where an AI coding agent permanently destroyed critical\
  \ project files through a sequence of git operations, and the systematic guardrails\u2026"
---

On 2026-07-22, an AI coding agent finished a six-phase implementation, merged cleanly to main, and then — during cleanup — permanently destroyed the plan document that had governed the entire build. The plan file was gone before anyone noticed it was missing. This is the account of how that happened, what made it hard to detect, and what the system looks like now.

---

## What Was Being Built

The feature in question added write capability to an existing Telegram-integrated vault system: seven new confirmation-gated tools (`save_draft`, `capture_idea`, `save_research_note`, `vault_create_file`, `vault_append_to_file`, `vault_update_frontmatter`, `vault_create_directory`), a shared validation and I/O layer in `vault_io.py`, and an extended confirmation UI that showed users full content previews before committing any write to disk. Nineteen tasks across six phases, each phase handled by a dispatched subagent, with independent re-validation and a commit after every phase before the next subagent was dispatched.

The execution was, by every measurable indicator, good. Every phase's ruff, pyright, bandit, and pytest checks passed on independent re-run. A branch-wide code review after all six phases found four real issues — three medium severity, one low — all fixed before merge. The feature shipped cleanly.

Then the worktree was removed.

---

## The Chain That Destroyed the File

AI coding agents working on isolated features typically use git worktrees: a separate checkout of the repository in a subdirectory, on a dedicated branch, so the main working tree is never disturbed. The plan file — the PRD that `prd-drafter` had written and `prd-executor` had consumed task by task across all six phases — lived inside that worktree at `.claude/worktrees/vault-write-capability/`. It was never committed. It was an untracked file.

Here is why that matters: git has no knowledge of untracked files. They don't appear in `git status` output in a way that blocks destructive operations. They don't get stashed. They are invisible to `git restore`. And critically, they are not protected by `git worktree remove` — unless you omit `--force`, in which case git will refuse to remove a worktree with modified *tracked* files, but will cheerfully proceed if the only files at risk are untracked. `git worktree remove --force` is the nuclear option, and it does exactly what it says.

The sequence that caused the loss was a small cascade. A stash pop landed on the wrong branch, creating unexpected state. `git restore` was run to clean things up — but without first auditing what untracked files were present, and `git restore` doesn't touch untracked files anyway, so the plan file survived that step. Then `git worktree remove --force` was run to finish cleanup, and that was the end of the plan file. The system review also notes a second loss: a system review document that had been in-progress in the same worktree was destroyed in the same operation.

No error. No warning. Silent deletion.

---

## The Convention That Already Existed (And Why It Failed)

There was already a rule for this. The development process had a convention: commit the PRD to the branch before beginning execution. If the plan file had been committed at any point during the six phases, it would have been part of the branch history, visible to git, and either safely merged to main or clearly present in the worktree's tracked state. `git worktree remove --force` would not have deleted it silently — or even if it had, the commit would have preserved it.

The convention existed. It was not followed. The reason it was not followed is the most instructive part of this incident: the convention lived only in memory. It was written nowhere that the agent's execution pipeline was required to check. It was not a pre-execute checklist item. It was not a mandatory step in the subagent dispatch protocol. It was a lesson someone had learned, held as implicit knowledge, and never encoded as an executable asset.

This is an extremely common failure mode in any system that relies on human memory as the enforcement mechanism for safety conventions. The rule exists; the rule is known; the rule fails at the moment it is needed because there is no friction between the operator and the dangerous action. The agent ran `git worktree remove --force` without anyone checking whether there were untracked files worth saving, because there was no step in the process that required that check.

---

## What Was Hard to Recover

The plan file could not be recovered. There is no git history for an untracked file that was never committed. No backup existed at the point of deletion. The system review written afterward notes this clearly and labels the analyzed plan document as "RECONSTRUCTED" — rebuilt from the execution report, the code review comments, and the shipped code itself.

That reconstruction is inherently incomplete and circular in a specific way: the execution report and code review were written *from* the plan, so reconstructing the plan from them produces something that matches what was executed, but cannot reveal gaps between what was planned and what was actually built. The system review scores execution fidelity at 9/10 against surviving evidence, but explicitly acknowledges that any "the executor followed the plan exactly" statement is partly circular when the plan is itself reconstructed from the executor's outputs.

Two things were lost that cannot be fully recovered: the original plan document as a historical artifact, and the ability to do a clean plan-versus-execution audit for this feature. The second loss is the more meaningful one for a system that treats those audits as a quality gate.

---

## The Guardrails Built Afterward

The response was not to write another memo about the convention. The response was to encode the convention in places that create real friction at the right moment.

The pre-execute checklist now includes a mandatory step: before beginning phase 1, commit the PRD to the feature branch. This is not advisory. The subagent dispatch protocol requires a commit reference to the plan file before a subagent can be dispatched. If that reference is absent, the orchestrator does not proceed.

The worktree cleanup sequence now includes an explicit untracked-file audit step before any `git worktree remove` invocation. The step runs `git status` in the worktree, lists untracked files, and requires a decision for each: commit it, copy it to a backup location, or explicitly acknowledge it as disposable. `--force` is not available as a shortcut past this step.

A vault backup was added as a second line of defense: plan files and system review documents are now copied to the vault (the Obsidian-backed note store that the vault-write feature itself was built to extend) as part of the completion sequence, before any worktree removal. This is deliberately redundant — it is not the primary safeguard but a fallback for the cases where the primary safeguard is bypassed.

---

## The Broader Principle

The incident is useful not because it involved an AI agent specifically, but because of what it reveals about how safety conventions degrade. An AI agent executing a sequence of git operations is not meaningfully different from a script executing the same sequence, or a tired engineer executing it on a Friday afternoon. In all three cases, the question is the same: is the safety check a thing that exists in someone's head, or is it a thing that exists in the process?

A lesson that lives only in memory has a half-life. It survives as long as the person who learned it is present, attentive, and remembers to apply it at precisely the moment it is needed. A lesson written into an executable asset — a checklist that blocks execution, a required commit reference, an audit step that cannot be skipped — survives regardless.

The vault-write feature shipped correctly. The plan that governed it no longer exists in its original form. Both of those facts are true, and the second one is the one that changed how this system works.
