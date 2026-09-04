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
  version: "1.10.0"
---

# Outstanding Board Workflow

The board is the durable, cross-session record of open work. It lives in the project memory directory as `project-outstanding-board.md`, with finished work in `project-outstanding-board-archive.md`.

This skill exists because of one specific, repeated failure: **a checkpoint reads as a terminator, so open items below or inside it get missed.** It happened enough times that guidance was not enough. The fix is structural, and it is the invariant below.

⚠️ This is **not** the harness's `TodoWrite` list. That is transient per-session scratch state. The board is durable and survives across sessions. Never conflate them.

## The Invariant

> **The MASTER LIST is the only place an open item may exist. A checkpoint is immutable history and carries zero unique information about open work.**

Three consequences, and all three are load-bearing:

1. **A checkpoint's "next actions" are master-list IDs, never prose.** Write `Next: #101, #104`. Never describe the work again in the checkpoint — a description is how an item ends up existing only there.
2. **Reciting the board means printing the 📺 PRINTABLE BOARD block**, a pre-rendered view of the master list kept in sync at every checkpoint. Checkpoints are read for context (decisions, corrections, evidence), never for inventory. The master list stays authoritative: when block and list disagree, the list wins and the block is stale.
3. **Because of 1 and 2, it is safe to stop reading after the latest checkpoint.** That is the point. The invariant buys the right to stop reading; without it, every recital needs a full end-to-end read, which has already been shown to fail on a long append-heavy file.

## Permanent IDs — Never Renumber

Every master-list item carries a **permanent ID** from one incrementing counter: `#101`, `#102`, `#103`. Assign it once. It belongs to that item forever, and it is **never reused** after the item closes. Table order is free to change; IDs are not.

**An ID is spent the moment it is ASSIGNED, not when the item closes.** ⛔ Never reuse a number, whatever became of the item that held it — closed, archived, merged into another item, or **rescinded before any work started**. Retirement is unconditional.

**The rescinded case is the one that looks reusable, and is not.** An ID handed out and withdrawn minutes later, for an item that never existed in any real sense, invites winding the counter back. ⛔ Do not. By the time you assign an ID you have usually already written it somewhere you are forbidden to edit — a checkpoint, an archive annotation, a memory file. Those records say *"`#NNN` opened"*. Reissue the number to a different item and every one of them silently starts pointing at the wrong work. The counter is cheap; three digits never run out. An ambiguous historical record is not cheap.

⚠️ **When you do rescind one, say so next to the counter** — the number, both dates, why it was withdrawn, and ⛔ do not reuse. That line is what stops a later session "reclaiming" an apparent gap in the sequence. A gap in the numbering is normal and needs no explanation; a gap someone tries to fill is the bug.

⭐ Recorded 2026-09-04. `#137` was opened for a doc write-up, and the user redirected the work into the re-opened `#136` instead. Claude retired `#137` rather than winding the counter back, because a dated checkpoint line already recorded it as opened. The user confirmed the call: *"never re-use an item number even if that number has been rescinded."*

**The counter lives on the board**, in the 🆔 block above the master list, as a `NEXT ID TO ASSIGN` line. Bump it the moment you assign one. ⛔ **Never derive the next ID by scanning the table.** Closed IDs are gone from it, so the highest open ID is not the highest assigned ID, and scanning would hand out a duplicate.

**Why the old scheme failed:** positional row numbers renumber whenever an item closes, and every reference written before that silently rots. The board was renumbered 13 times in 6 days. `MEMORY.md` ended up pointing at "board row 8" for an item that had become row 7. A cross-reference in a memory file, a checkpoint, or a conversation now stays correct forever. (The user proposed this on 2026-08-09; the numbering starts at 101 so an ID never looks like a positional index.)

**Old references still say "row N".** Every checkpoint, archive entry, and memory file written on or before 2026-08-09 uses the positional scheme. ⛔ **Do not back-propagate IDs into them** — they are dated records, and rewriting them falsifies the record. The board carries a translation table for the final positional numbering, plus a frozen renumber log for resolving older references. Translate on read; write only IDs from now on.

**This section is the authoritative statement of the rule. The board holds only the data.** That split is deliberate — a rule written in 2 places drifts. On the board you will find the counter, the translation table, and the frozen renumber log, plus a 2-line guard so a session that never loaded this skill still cannot renumber or reuse by accident. Everything else about the scheme belongs here. Fix a conflict here, not there.

**Starting a fresh board:** open the counter at `#101`. Three digits keep an ID from ever reading as a positional index, and the leading `1` makes `#101` visibly not "item 1".

## Procedure A — OPEN the Board

Run this on "resume", "show the board", "what's outstanding", or before answering any question about remaining work.

> ⚡ **STOP AFTER STEP 1 for a bare "show the board" / "what's outstanding" / "what's left".** Those ask for the inventory, and step 1 is the whole answer. **Steps 2–5 run only when you are about to WORK the board** — a "resume", a checkpoint, or any turn that will act on an item. Running steps 3–5 to answer a display request is exactly the waste this design removes.

1. **"Show the board" is a VERBATIM PRINT, not a read of the board.** The board keeps a pre-rendered 📺 PRINTABLE BOARD block, regenerated at every checkpoint. Print it and stop — one command, ~1.3 KB, no summarising:

   ```bash
   cd <project memory dir>
   awk '/BOARD-TABLE:BEGIN/{f=1;next} /BOARD-TABLE:END/{exit} f' project-outstanding-board.md
   ```

   ⛔ **Do NOT read the master list, a checkpoint, or `MEMORY.md` to answer "show the board".** The block already carries the item count, every row, the blocked-on-user line, and the `Next:` pointer. ⛔ **Do NOT dump a line range from the board** — it is ~490 KB and a single master-list row runs to 7 KB on one line, so even `sed -n '29,100p'` overruns the tool's output cap and gets truncated to a file.

   **If the block is missing or stale**, regenerate it with Procedure B step 4 before answering, and say in one line that you did. Staleness shows as a count mismatch:

   ```bash
   grep -c '^| \*\*#' project-outstanding-board.md     # master-list rows; must equal the block's count
   ```

2. **Anything deeper than a recital reads the MASTER LIST**, which stays authoritative for detail. Pull rows without dumping a range:

   ```bash
   awk '/^\| \*\*#/' project-outstanding-board.md | cut -c1-400
   grep -n 'NEXT ID TO ASSIGN' project-outstanding-board.md
   ```

   ⚠️ **A `cut` slice is a truncation, not a summary** — a 4 KB cell's operative qualifier can sit past the cut. Never paraphrase a sliced cell into a claim; widen the slice for the row you actually need.

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

#**This section covers the BOARD half only.** The general rule — Claude's own memory, index, board and scratchpad upkeep is done automatically, never announced, never offered as a choice — lives in the user's global `CLAUDE.md` under Working Method, because it applies when no board is in sight (editing a doc, verifying anchors, re-rendering a diagram) and `CLAUDE.md` is always loaded whereas this skill is not. ⛔ Do not restate that rule here; a rule written in 2 places drifts.

### Two Different Requests

Displaying the board and recapping the session are separate asks. Do not merge them.

| The user says | You answer with |
|---|---|
| **"resume"** (bare) | **The next concrete action itself.** Not an inventory — go to the item the latest checkpoint's `Next:` line names, and open with the step that item is actually sitting on. |
| "show the board", "what's outstanding", "what's left" | The master-list table. Nothing else. |
| "where were we?", "what did we do last session?", "catch me up" | The latest checkpoint's context. Add the table if it helps. |

⚠️ **"resume" means CONTINUE THE WORK, not "inventory the work."** Recorded 2026-08-30, after a bare "resume" was answered with the 9-row master list: *"my original prompt was 'resume', not 'show the board'. So, why didn't you bring me right to the step inside #114?"* The checkpoint's `Next:` line exists precisely to answer "resume" in one hop — read it, then go do that. Recite the table only if the `Next:` line is missing or names something already closed, or if the user asks for the list.

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
2. **Add master-list rows FIRST, before writing any checkpoint prose.** Every new open item gets a row now, with the next ID off the board's counter. Bump the counter in the same edit. This ordering is the whole mechanism: prose written first is prose that becomes the item's only home.
3. **Close finished items** — delete the row, move the detail to the archive. The ID goes with it and is retired, never reused (as is any ID that was assigned and then rescinded — see "Permanent IDs"). Do not leave a `✅` row on the live board; that is what makes the list long enough to bury things.
   - **A standing reminder is a finished item too, and it needs one extra step.** A reminder block carries *imperatives* ("raise it unprompted", "raise it every session", "on every resume until it is done"). Appending a `✅ COMPLETE` line to the bottom does **not** cancel them — the next session reads the imperative and obeys it. **Move the whole block to the archive** the moment the work completes, and leave a one-line pointer on the board saying it is retired and must not be raised again. See "Retiring a Standing Reminder" below.
4. **Regenerate the 📺 PRINTABLE BOARD block.** ⭐ **This is what makes "show the board" cheap** — it moves the whole cost of rendering into the checkpoint, where it belongs. Rewrite the block between the 2 delimiters so it matches the master list exactly: the item count in the heading, one row per open item, the blocked-on-user line, and a `Next:` line matching the checkpoint's. Keep each row on one line and under ~150 characters — this block is a VIEW, and detail belongs in the master-list row. ⛔ Never edit a row here without editing its master-list row in the same pass.

   ⚠️ **Keep the delimiter strings out of surrounding prose.** The extractor matches the first occurrence, so a copy of the marker in a nearby paragraph or code block makes it match its own documentation and print the instructions instead of the table. That is why the board file points at this skill for the command rather than repeating it.

5. **Write the checkpoint as history.** What happened, what was decided, what was corrected, what the evidence was. Next-actions are IDs only.
6. **Sweep `MEMORY.md`'s Active Work** so it matches the master list. Move closed entries to `MEMORY-CLOSED.md` with the evidence for closing each.
7. **Record doc baselines** if docs changed — hash, line count, snapshot path.
8. **Run the invariant self-check below.** This is a required step.

### The Invariant Self-Check

> Read the checkpoint you just wrote. **Does any sentence describe work that has no master-list row?**

If yes, you have just recreated the original failure. Add the row before finishing.

### Re-opening a Closed Item

A closed item can come back — usually because closing it orphaned work that had no other home. Keep its ID and its history; ⛔ do not mint a new number for the continuation.

1. **Restore ONE row to the master list, under the original ID**, and write it as a live spec: what is done, what is now open. ⛔ Do not paste the archived text back verbatim — it was written to describe a finished thing.
2. **Leave the archive entry in place and annotate it as superseded**, naming the live row as the authority. ⛔ Never delete it; it is a dated record, and it usually stays accurate about the part that really did finish.
3. **Say in the row why it re-opened.** That reason is the thing a later session needs, and it is the thing most easily lost.
4. **⛔ Do not split the finished part from the continuation across 2 items.** The evidence and the work it feeds are one body; separating them parks the evidence in an archive nobody reads.

⭐ Recorded 2026-09-04, from `#136`: an experiment closed cleanly, but the write-up it produced fitted no other open item. Claude first opened a new item for the write-up; the user redirected it back into `#136` and was right — the experiment and its finding are one body of work.

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


