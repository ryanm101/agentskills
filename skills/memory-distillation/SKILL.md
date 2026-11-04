---
name: memory-distillation
description: Mines a project's raw session transcripts and existing memory for recurring errors, process feedback, and tool quirks that were never written down, writes the project-specific lessons into that project's memory store, then runs a second pass to generalize and promote the reusable subset into a permanent cross-project memory store. Use when asked to review past sessions/transcripts for patterns worth remembering, to do a "memory retrospective," to bootstrap or refresh permanent agent memory, or after a notably painful session that should never repeat the same mistake.
---

# Memory Distillation

## Overview

A session accumulates far more operational knowledge than ever gets written
down: tool gotchas discovered the hard way, environment quirks, corrections
the user gave once and shouldn't have to give again. Most of it dies with
the context window. This skill is a repeatable retrospective over the raw
transcripts and the existing memory store that pulls that knowledge out,
separates what's genuinely new from what a committed doc already covers,
and files it in the right place.

Two passes, two audiences, two stores:

- **Pass 1 — project memory.** Lessons tied to this specific repo/user
  relationship: collaboration style, this project's tool-setup quirks,
  roadmap/status nuance. Lives at this project's own memory path.
- **Pass 2 — permanent store.** The generalizable subset of Pass 1 — the
  mechanism stripped of this project's specific nouns, restated so it would
  save time in an unrelated project too. Lives in a durable, cross-project
  location outside any single repo.

The two-pass split matters: a lesson that's obviously true but only true
*here* (a specific function name, a specific binary's quirk) pollutes a
permanent store if promoted as-is, while a lesson that's genuinely general
(a tool's known failure mode, a collaboration pattern) wastes its value if
left stranded in one project's memory where no other project will ever read
it.

## When to Use

- Explicit request to review transcripts/sessions and save what was learned
- "Turn this into permanent memory" / "do a memory retrospective"
- Bootstrapping memory for a project that already has a long session history
  but no memory store yet
- After a single unusually costly session (many retries, a hard-won
  debugging breakthrough) — run a lighter single-session version of Pass 1
  only, skip the multi-transcript survey in Step 1

**When NOT to use:**

- A routine end-of-task save of one clear correction or fact — that's
  ordinary memory-writing; this skill's transcript-mining machinery is
  overkill for a single data point.
- The lesson is already fully captured in a committed project doc
  (`CLAUDE.md`, `AGENTS.md`, `README.md`, or equivalent) — point there
  instead of duplicating it into memory.

## Inputs to Locate First

1. **Committed knowledge docs** — `CLAUDE.md`, `AGENTS.md`, `README.md`,
   `CONTRIBUTING.md`, or equivalent. Read these FIRST, in full. They're the
   one thing every future agent sees regardless of memory; nothing already
   stated there belongs in memory too (see Step 3).
2. **The project's existing memory store** — its `MEMORY.md` index and the
   files it links to. Read before writing anything; this pass extends it,
   it does not start over or re-derive what's already there.
3. **Session transcripts for this project** — the raw conversation logs for
   this working directory. List them and check size/line-count on each
   before deciding how to process it (Step 1).
4. **Informal in-repo "lessons" files** a past session already wrote (a
   `note.md`, `LEARNINGS.md`, a dated findings doc). These are often already
   a first draft of exactly what this skill produces — read them and treat
   their content as already-captured, but still mine the raw transcripts
   for whatever never made it into those docs.
5. **The permanent cross-project store's own `MEMORY.md`** (Pass 2's
   destination) — read before writing, for the same reason as #2: this
   skill may have already run against a different project.

## The Process

```
Memory distillation checklist:
- [ ] Step 0: SURVEY   — found committed docs, existing memory, transcripts,
                          in-repo lessons files, and the permanent store
- [ ] Step 1: SCALE    — sized each transcript; chose direct read, scripted
                          extraction, or delegated mining per file
- [ ] Step 2: MINE     — extracted corrections, repeated errors, tool
                          quirks, and confirmed preferences, with quotes
- [ ] Step 3: DEDUPE   — dropped anything already in committed docs or
                          existing memory; merged near-duplicates
- [ ] Step 4: WRITE PROJECT MEMORY   — correct frontmatter/type, linked,
                          indexed
- [ ] Step 5: GENERALIZE — asked, per finding, "would this help in an
                          unrelated project?"; promoted the "yes" subset
- [ ] Step 6: WRITE PERMANENT MEMORY — same format, in the durable store,
                          indexed, deduped against what's already there
- [ ] Step 7: VERIFY   — read back what was actually written, not a
                          subagent's summary of it
- [ ] Step 8: REPORT   — short summary to the user of what changed and why
```

### Step 0: Survey

Read every doc in "Inputs to Locate First" before touching a single
transcript. The single biggest risk in this whole skill is re-deriving or
duplicating something a past session already wrote down somewhere durable —
Step 0 is the only defense against that, and it's cheap compared to mining
raw logs.

### Step 1: Scale — decide how to process each transcript

Transcripts are typically one JSON event per line. Before reading any of
them, check size and line count on every file.

- **Small** (low thousands of lines / under ~1MB): read or grep directly.
- **Large** (tens of thousands of lines / multiple MB): do **not** load it
  into your own context with a generic file-read tool — it will blow the
  context budget before covering a fraction of the file. Instead:
  - Write a small streaming script (`python3`/`jq`) that filters line by
    line for the Step 2 patterns and emits only the matches, keeping your
    own context clean, **or**
  - Dispatch a **fresh, non-fork subagent** per large file to run that
    extraction and report back a condensed, quote-backed bullet list.
- Multiple large transcripts: dispatch one subagent per file **in
  parallel**, not sequentially — this is pure independent I/O-bound
  extraction with no cross-file dependency to serialize on.
- Give each dispatched subagent an explicit, narrow brief: what patterns to
  extract, what format to report in, a word/line budget, and — critically —
  that its job is to **report, not to act** (see Step 7 for why this needs
  saying explicitly).

### Step 2: Mine — what to look for in each transcript

Four categories, roughly in order of value:

1. **Explicit process feedback** — the user telling the agent *how to
   work*, not *what to build*. Corrections ("no", "don't", "stop",
   "actually", "that's wrong") **and** confirmations ("yes exactly",
   "perfect", "confirmed") are both signal. A confirmation validates a
   non-obvious choice that would otherwise look arbitrary to a future
   session reading the code cold.
2. **Recurring tool/environment friction** — the same tool error, timeout,
   or wrong assumption hit three or more times. A one-off error isn't worth
   memory; a pattern is.
3. **Self-corrections and root-caused dead ends** — assistant text admitting
   a wrong turn, *especially* when it explains the root cause rather than
   just that something failed. The root cause is the reusable part.
4. **Anything explicitly flagged to remember** — "write this down",
   "remember this", "make this a note.md for next time" — treat as a direct
   instruction and give it top priority regardless of how it was phrased.

For each candidate finding, capture: a short verbatim quote (the evidence),
what it was correcting or describing, and a provisional tag — project-fact,
process-feedback, or tool-quirk. Don't round a single occurrence up to "a
pattern"; note recurrence counts honestly.

### Step 3: Deduplicate

Before writing anything:

- Drop anything already stated in the committed docs found in Step 0.
  Memory should hold what docs structurally can't — collaboration style,
  process feedback, cross-session tool gotchas — not re-document the repo.
- Drop anything already in the existing memory store (check by reading its
  `MEMORY.md` and the files it links to, not just by guessing from
  filenames).
- Merge near-duplicate findings across transcripts/subagents into one
  entry. If a lesson recurred across multiple sessions, say so in the entry
  — repetition is itself signal for how much weight it deserves, not
  something to silently collapse into the first instance seen.

### Step 4: Write project memory

Use the memory system's actual established format exactly — don't invent
one. In a Claude Code environment with auto-memory configured, that means:
frontmatter with `name`, `description`, and `metadata.type` (one of
`user`/`feedback`/`project`/`reference`); a body that, for `feedback` and
`project` types, states the rule/fact then a **Why:** line and a **How to
apply:** line; `[[other-entry-name]]` links between related memories; and a
one-line addition to that store's `MEMORY.md` index (under ~150 characters).
If this environment has no established memory format, ask rather than
inventing one — don't guess at a schema a future session won't recognize.

### Step 5: Generalize — the second pass

For every project-memory entry just written, ask explicitly: *if this
user/agent pair were working on a completely unrelated project — different
language, different domain, different tools — would knowing this in advance
still save time or prevent an error?*

- **Yes, once stripped of this repo's specific nouns** → promote to the
  permanent store (Step 6). Restate the mechanism generically — e.g. "this
  project's `SDL2_ttf` build mismatch" becomes "installing a library
  through a different package manager than an existing dependency already
  resolves through can link it against a mismatched system version."
- **No, inherently tied to this codebase, binary, or domain** → stays in
  project memory only.
- A single finding can split into both: write the generic mechanism to the
  permanent store, and — if still worth a pointer — keep a short
  project-specific entry in project memory that links to it, so a future
  session in *this* project doesn't have to go find the general version to
  get the local specifics.

### Step 6: Write permanent memory

Same format discipline as Step 4, written into the durable cross-project
store. Read that store's own `MEMORY.md` first (Step 0, input #5) so this
pass extends prior runs of this skill against other projects rather than
duplicating them. If the store doesn't exist yet, confirm the intended path
with the user before creating it — don't assume a default location is
correct for every environment.

### Step 7: Verify — don't trust your own subagents' summaries

The step most likely to be skipped, and the one most likely to matter:

- If any mining or writing was delegated, **read the files actually
  written** (or the actual diff/output), not just the subagent's final
  summary of what it believes it did. A summary describes intent, not
  necessarily outcome.
- This risk is specifically elevated for **forks**: a fork inherits the
  entire parent conversation, including any broader goal stated earlier in
  it (e.g. "then store this as permanent memory" said three turns before
  the fork was dispatched). Given both that inherited goal and a narrower
  literal instruction in its own dispatch prompt ("just report back, don't
  write anything"), a fork can reasonably act on the larger inherited goal
  instead of the smaller one it was actually asked to do in isolation — and
  may not flag that it did so. For any step where the instruction is "only
  do X, not the eventual Y," prefer a **fresh, context-free subagent**
  over a fork; reserve forks for steps where "it might reasonably do more
  than literally asked" is an acceptable outcome.
- Check for bloat while verifying: near-duplicate entries, or entries that
  just restate something trivially derivable from reading the code or `git
  log`/`git blame` directly. Cut those before they get indexed.

### Step 8: Report

Tell the user what was added — counts and one-line summaries, not full file
dumps — and where (both store paths). Flag anything cut for being redundant
with existing docs or memory; that's a useful signal that the committed
docs are already doing their job and don't need a memory-layer duplicate.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll just read the whole multi-MB transcript directly, it's fine" | It will blow the context budget before covering a fraction of the file. Script the extraction or delegate it (Step 1). |
| "The subagent said it's done, that's good enough" | Summaries describe intent. Read what was actually written before reporting completion (Step 7). |
| "This fact is interesting, I'll save it" | Interesting isn't the bar. The bar: would *not* knowing this cost a future session real time or a real error, and is it *not* already written down somewhere durable? |
| "It's generic enough, skip the project-memory version and just write the permanent one" | Keep both when relevant (Step 5) — a future session in *this* project still benefits from a local pointer even once the general lesson lives elsewhere. |
| "One big file is simpler than many small ones" | The established memory format expects one file per entry plus an index, so each lesson can be independently updated or removed later. Don't flatten that for convenience. |
| "A fork is fine for a read-only mining task" | Forks inherit the whole conversation and can act on goals beyond the literal instruction (Step 7). Use a fresh subagent when the instruction must not be exceeded. |

## Red Flags

- Loaded a multi-MB transcript directly into your own context "just to check something quickly"
- Wrote a memory entry that only restates something already in the project's committed docs
- Reported a distillation pass "complete" without reading the files a delegated subagent actually wrote
- Dispatched a fork for a task whose instruction explicitly required it to *not* exceed its literal scope
- A `MEMORY.md` index entry over ~150 characters, or missing for a file that exists on disk
- Promoted a repo-specific fact (a function name, a file path, a particular binary's quirk) into the permanent store without generalizing it first

## Verification

- [ ] Every new project-memory file is linked from that store's `MEMORY.md`
- [ ] Every new permanent-memory file is linked from the permanent store's `MEMORY.md`
- [ ] Nothing written duplicates content already in the project's committed docs
- [ ] Every delegated subagent's or fork's actual output was read and checked, not just its summary
- [ ] Every permanent-store entry is stripped of repo-specific specifics and states a generic mechanism
- [ ] Recurrence counts and confidence are stated honestly, not rounded up to "a pattern" from one instance
- [ ] The user received a short, concrete summary of what was added and where — not a restatement of this checklist
