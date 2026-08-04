---
name: outstanding-board-workflow
description: >
  The read/write protocol for the project's outstanding-work board
  (`project-outstanding-board.md` in the project memory directory) — the durable,
  cross-session record of what is still open. Use this skill whenever you open the board
  or add to it. Trigger on: "resume", "show the board", "what's outstanding", "what's
  left", "where were we", "checkpoint", "let's checkpoint", "wrap up", "end of session",
  "save state", "housekeeping on the board", or any turn where you are about to write a
  checkpoint or recite open items. Covers the two procedures (OPEN the board, CLOSE with a
  checkpoint), the master-list invariant that stops open items being buried inside
  checkpoint prose, live-state re-verification before reciting infra, and reconciliation
  against MEMORY.md. Do NOT use this for the harness's ephemeral per-session TodoWrite
  list — that is unrelated transient state, not the board.
license: Apache-2.0
metadata:
  author: Vincent Yin
  version: "1.0.0"
---

# Outstanding Board Workflow

The board is the durable, cross-session record of open work. It lives in the project memory directory as `project-outstanding-board.md`, with finished work in `project-outstanding-board-archive.md`.

This skill exists because of one specific, repeated failure: **a checkpoint reads as a terminator, so open items below or inside it get missed.** It happened enough times that guidance was not enough. The fix is structural, and it is the invariant below.

⚠️ This is **not** the harness's `TodoWrite` list. That is transient per-session scratch state. The board is durable and survives across sessions. Never conflate them.

## The Invariant

> **The MASTER LIST is the only place an open item may exist. A checkpoint is immutable history and carries zero unique information about open work.**

Three consequences, and all three are load-bearing:

1. **A checkpoint's "next actions" are master-list row numbers, never prose.** Write `Next: items 1 and 2`. Never describe the work again in the checkpoint — a description is how an item ends up existing only there.
2. **Reciting the board means reciting the master list.** Checkpoints are read for context (decisions, corrections, evidence), never for inventory.
3. **Because of 1 and 2, it is safe to stop reading after the latest checkpoint.** That is the point. The invariant buys the right to stop reading; without it, every recital needs a full end-to-end read, which has already been shown to fail on a long append-heavy file.

## Procedure A — OPEN the Board

Run this on "resume", "show the board", "what's outstanding", or before answering any question about remaining work.

1. **Read `MEMORY.md`, then the board.**
2. **Read the MASTER LIST table. That is the inventory.** Recite from it. If the board has no master list, build one before answering — sweep the whole file plus `MEMORY.md`'s Active Work section and enumerate every open item into a table at the top.
3. **Re-verify any live-state claim before repeating it.** Board labels about infrastructure go stale. Read-only checks are free, so run them: `gcloud compute instances list`, `addresses list`, `networks list`, `firewall-rules list`, and whatever else the board asserts. Never recite a resource list you have not just confirmed.
4. **Read the latest checkpoint for CONTEXT ONLY** — decisions to honor, corrections the user made, evidence pointers. Do not mine it for items. If you find an open item there that has no master-list row, that is a defect: add the row.
5. **Reconcile the master list against `MEMORY.md`'s Active Work section.** Any item present in one and absent from the other is a bug. Fix both. Also check `MEMORY-CLOSED.md` before assuming a topic has no prior memory.

### Verify a Status Before Relaying It

A status marker is a claim, not a fact. Two known traps:

- **A file can contradict itself.** The newest specific claim wins. Check dates before believing any `⬜`, `OPEN`, or `IN PROGRESS`, including one inside a table cell.
- **A memory file's status can lag its own conclusion.** Read the file's description line and its newest entry, not just the index hook in `MEMORY.md`.

When a contradiction turns up, fix **all** copies of the stale claim, not only the one that prompted the check.

## Procedure B — CLOSE with a Checkpoint

Run this on "checkpoint", "wrap up", "end of session", or before a long break.

1. **Establish what changed** this session — work done, decisions made, corrections received.
2. **Add master-list rows FIRST, before writing any checkpoint prose.** Every new open item gets a row now. This ordering is the whole mechanism: prose written first is prose that becomes the item's only home.
3. **Close finished items** — delete the row, move the detail to the archive. Do not leave a `✅` row on the live board; that is what makes the list long enough to bury things.
4. **Write the checkpoint as history.** What happened, what was decided, what was corrected, what the evidence was. Next-actions are row numbers only.
5. **Sweep `MEMORY.md`'s Active Work** so it matches the master list. Move closed entries to `MEMORY-CLOSED.md` with the evidence for closing each.
6. **Record doc baselines** if docs changed — hash, line count, snapshot path.
7. **Run the invariant self-check below.** This is a required step.

### The Invariant Self-Check

> Read the checkpoint you just wrote. **Does any sentence describe work that has no master-list row?**

If yes, you have just recreated the original failure. Add the row before finishing.

## Never Rewrite a Checkpoint

A checkpoint is a historical record. When a later decision supersedes it, **leave the old text alone** — rewriting it falsifies the record. Instead, note the supersession in the master list or in the closed-items block near it, naming what is stale and why.

This applies to renamed resources too. Text describing what previously existed keeps the old names.

## Why This Skill Exists

Recorded so the reasoning survives:

- The board grew to 5 nested superseded checkpoints. Each ended with a forward-looking "next on resume" list, which made every checkpoint look like an inventory.
- On 2026-08-03 the user asked to "show the board." The whole file was read, and roughly 6 open items were still dropped — some lived only inside superseded checkpoint prose, and one (`Docs Q&A Agent`) was never on the board at all, only in `MEMORY.md`.
- Separately, `MEMORY.md`'s Active Work list carried 3 entries whose own files said CLOSED.
- The always-loaded memory `feedback-check-outstanding-board-first-on-resume` had described the board as "the authoritative latest checkpoint", which trained the stop-reading behavior directly. It was repointed at the master list in the same change that created this skill.

Related: `feedback-verify-stale-status-claims-in-board` (the staleness half), `feedback-check-outstanding-board-first-on-resume` (the resume-order half).

