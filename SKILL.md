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
  checkpoint prose, when live-state re-verification is and is not warranted,
  reconciliation against MEMORY.md, and the no-rewrite rule for dated records
  (checkpoints, memory files, standing reminders).
  Do NOT use this for the harness's ephemeral per-session TodoWrite
  list — that is unrelated transient state, not the board.
license: Apache-2.0
metadata:
  author: Vincent Yin
  version: "1.5.0"
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
3. **Do NOT re-verify live cloud state by default.** Reciting the board is a read of the board, not an audit of the cloud. Skip the `gcloud` sweep unless one of these holds:
   - The user asks for it.
   - The next action you are about to take depends on a resource actually existing.
   - You are about to make a positive claim about live infrastructure that goes beyond "the board says X" — in that case either verify it or attribute it to the board.

   When a later task does need current infra state, set the check up then. (User's call, 2026-08-09: an unconditional sweep on every board display is wasted work.)
4. **Read the latest checkpoint for CONTEXT ONLY** — decisions to honor, corrections the user made, evidence pointers. Do not mine it for items. If you find an open item there that has no master-list row, that is a defect: add the row. ⚠️ **This read informs you; it is not output.** Do not relay it in a board recital — see "What a Recital Actually Shows" below.
5. **Reconcile the master list against `MEMORY.md`'s Active Work section.** Any item present in one and absent from the other is a bug. Fix both. Also check `MEMORY-CLOSED.md` before assuming a topic has no prior memory.

### What a Recital Actually Shows

A board recital answers one question: **what is still open?** The master list answers it. Almost everything else in the file is machinery for maintaining the board, and reciting machinery buries the answer.

**Show the master-list rows. That is the whole answer.** At most, add one short line naming which rows are blocked on the user.

**Do not show** (internal, keep it to yourself):

- **The latest checkpoint's contents.** No session recap, no "here's what changed last time", no digest of decisions to honor. See "Two Different Requests" below.
- **Doc baselines** — md5 hashes, line counts, snapshot paths. These exist so *you* can detect a hand-edit the user never mentioned. They are an instrument, not news. Surface one **only** when it is actionable for them: drift turned up that they did not make, or they asked.
- **Bookkeeping** — renumbering history, reconciliation notes, archive pointers, master-list row counts as a statistic.
- **Closed items.** A closed item has no claim on their attention. That includes retired standing reminders.

⚠️ Recorded 2026-08-09, across 3 rounds of the same complaint. The user got a retired standing reminder, then a line of md5 hashes, and asked of each: *"Why do you need to remind me that at all? That's your internal housekeeping which I don't need to know."* They then cut checkpoint context too: *"If I only ask you to 'show the board', don't drag along the old baggage."* **The test is whether the line could change what they do next.** If not, it stays in the file where you found it.

#### Two Different Requests

Displaying the board and recapping the session are separate asks. Do not merge them.

| The user says | You answer with |
|---|---|
| "show the board", "what's outstanding", "what's left" | The master-list table. Nothing else. |
| "where were we?", "what did we do last session?", "catch me up" | The latest checkpoint's context. Add the table if it helps. |

Both phrasings load this skill, so the trigger list is not the discriminator. Read the request itself.

⚠️ **You still READ the checkpoint on every board open** — Procedure A step 4 is unchanged. Reading it is how you catch an open item that has no master-list row. That is a defect check, and **its output is a new row in the table, not a paragraph for the user.** Read it; do not relay it. Because any defect you find becomes a row, the table stays a complete answer on its own.

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
   - **A standing reminder is a finished item too, and it needs one extra step.** A reminder block carries *imperatives* ("raise it unprompted", "raise it every session", "on every resume until it is done"). Appending a `✅ COMPLETE` line to the bottom does **not** cancel them — the next session reads the imperative and obeys it. **Move the whole block to the archive** the moment the work completes, and leave a one-line pointer on the board saying it is retired and must not be raised again. See "Retiring a Standing Reminder" below.
4. **Write the checkpoint as history.** What happened, what was decided, what was corrected, what the evidence was. Next-actions are row numbers only.
5. **Sweep `MEMORY.md`'s Active Work** so it matches the master list. Move closed entries to `MEMORY-CLOSED.md` with the evidence for closing each.
6. **Record doc baselines** if docs changed — hash, line count, snapshot path.
7. **Run the invariant self-check below.** This is a required step.

### The Invariant Self-Check

> Read the checkpoint you just wrote. **Does any sentence describe work that has no master-list row?**

If yes, you have just recreated the original failure. Add the row before finishing.

### Retiring a Standing Reminder

A standing reminder is the one board construct that tells a **future** session to speak up unprompted. That makes it the one construct that keeps firing after the work is done.

**The failure it produces:** the reminder completes, someone appends `✅ FULLY COMPLETE` to the bottom of the block, and the block stays in the OPEN ITEMS section — still headed *"… IS DUE"*, still containing *"raise it unprompted … on every resume until it is done"*. Every later session reads the live imperative near the top, obeys it, and re-raises a closed item. The `✅` line at the bottom does not stop this, because the imperative is not a status field; it is an instruction.

**The fix, when the work completes:**

1. **Move the entire block to the archive, verbatim.** This is a relocation, not a rewrite, so the no-rewrite rule is satisfied — do not edit a word of the block itself.
2. **Head the archived copy with a warning** that its imperatives are spent, naming them, so a reader who lands there does not act on them.
3. **Leave a one-line pointer on the live board** — retired, moved, and ⛔ do not raise again, do not restore.
4. **Fix any pointer that referenced the block's position** ("the block below"). Those live in guidance text, which is editable in place.

**Why relocation and not annotation:** position is what makes a reminder fire. A block sitting in OPEN ITEMS reads as open, whatever its bottom line says. Only moving it out changes what the next session does.

⚠️ Recorded 2026-08-09, when the user asked *"Why do you keep reminding me of the Phase 9 teardown … It's ancient history by now."* The teardown finished 2026-08-08 and the block had carried a closing `✅✅ FULLY COMPLETE` line ever since — and sessions still raised it, because the *"Raise it unprompted"* line 22 lines above was never neutralized.

## Never Rewrite a Dated Record — "the no-rewrite rule"

**Cite this section by that name.** It is the only place the name is defined, so a reference to "the no-rewrite rule" always resolves here.

A dated record says what was true, or believed, on a given date. When a later fact supersedes it, **leave the old text alone** — rewriting it falsifies the record. Append the correction instead, dated, naming what is now stale and why.

**Scope — all 3 kinds of dated record:**

1. **Checkpoints on the board.** Note the supersession in the master list, or in the closed-items block near the checkpoint.
2. **Memory files** — running notes, learning tasks, plans. Put the correction in a block at the top of the body, above the original text.
3. **Standing reminders and other dated blocks** inside the board.

Renamed resources follow the same principle. Text describing what previously existed keeps the old names.

**What the rule does NOT cover.** Guidance text is not a record. The board's header and invariant block, a skill's own instructions, and `MEMORY.md`'s index hooks all describe how things are *now*, so edit them in place and do not append a correction. The test is whether the text carries a date or describes a past state.

⚠️ **Do not cite this rule for anything outside that scope.** It was invoked for months as unnamed shorthand, drifting past what any written source said, until the user challenged the citation on 2026-08-09 and found the name existed nowhere. Naming it here is the fix. Keep it honest: if a case is not covered above, say what you are doing and why, without borrowing this rule's authority.

## Why This Skill Exists

Recorded so the reasoning survives:

- The board grew to 5 nested superseded checkpoints. Each ended with a forward-looking "next on resume" list, which made every checkpoint look like an inventory.
- On 2026-08-03 the user asked to "show the board." The whole file was read, and roughly 6 open items were still dropped — some lived only inside superseded checkpoint prose, and one (`Docs Q&A Agent`) was never on the board at all, only in `MEMORY.md`.
- Separately, `MEMORY.md`'s Active Work list carried 3 entries whose own files said CLOSED.
- The always-loaded memory `feedback-check-outstanding-board-first-on-resume` had described the board as "the authoritative latest checkpoint", which trained the stop-reading behavior directly. It was repointed at the master list in the same change that created this skill.

Related: `feedback-verify-stale-status-claims-in-board` (the staleness half), `feedback-check-outstanding-board-first-on-resume` (the resume-order half).

