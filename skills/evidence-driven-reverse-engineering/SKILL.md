---
name: evidence-driven-reverse-engineering
description: Applies disciplined, evidence-based rigor to reverse engineering any opaque target — binaries, protocols, file formats, obfuscated code, firmware, or undocumented systems. Use when reconstructing unknown behavior from an artifact rather than from source or docs, when a wrong inference would be expensive to discover later (an exploit that doesn't fire, a driver that bricks hardware, a protocol replay that corrupts state), or any time you're about to promote "this looks like what it does" into a claim you'll report or act on.
---

# Evidence-Driven Reverse Engineering

## Overview

Reverse engineering has no spec to check against — the artifact itself is the only ground truth, and it is frequently optimized, stripped, obfuscated, or actively hostile to inspection. That makes the single most common failure mode **plausible pseudocode mistaken for confirmed behavior**: a decompiler produces a clean-looking function, a symbol name sounds right, a string nearby seems to confirm it, and the claim quietly hardens into fact without ever being tested against a second, independent source of evidence.

This skill borrows its posture from three sibling skills and fuses them for the RE context:

- **From `doubt-driven-development`**: every non-trivial claim gets an adversarial pass before it stands — here the adversary isn't "a fresh reviewer reading your code," it's "an alternative explanation for the same bytes."
- **From `source-driven-development`**: every claim needs a citable source — here the source is an address, offset, instruction, string, xref, or captured trace, not a doc URL, but the discipline of "cite it or flag it unverified" is identical.
- **From `context-engineering`**: RE sessions generate huge, disposable artifact dumps (disassembly, strings, packet captures) that will blow the context budget and vanish on compaction unless the *conclusions* are deliberately extracted into a persistent ledger.

The result is a loop built specifically for working backward from an artifact: form a falsifiable hypothesis, corroborate it from independent channels, label its confidence honestly, doubt it before it drives anything consequential, and keep a ledger that survives the session.

## Requirements

This skill assumes two dependencies are present before Step 1 begins:

1. **REA** (https://github.com/morluto/rea) — supplies the static/dynamic analysis tools (`mcp__rea__*`) this skill cites as evidence channels.
2. **agent-skills** (https://github.com/addyosmani/agent-skills) — supplies the sibling skills this one fuses together (`doubt-driven-development`, `source-driven-development`, `context-engineering`) and the `reverse-engineer-anything` toolbox skill.

Check for both before relying on them:

- REA: confirm an `mcp__rea__*` tool is available (e.g. via `ToolSearch` or the tool listing). If absent, tell the user: "This skill requires REA — install it from https://github.com/morluto/rea and configure it as an MCP server, then retry."
- agent-skills: confirm the sibling skills above are listed as available skills. If absent, tell the user: "This skill requires the agent-skills collection — install it from https://github.com/addyosmani/agent-skills, then retry."

Do not silently fall back to ad-hoc analysis in place of a missing dependency — surface the gap and stop.

## When to Use

- Reconstructing the behavior of a binary, library, protocol, file format, or API with no authoritative documentation
- Decompiling or disassembling code and reporting what it does
- Reverse engineering an undocumented wire protocol, save format, or on-disk layout from captured samples
- Working out what obfuscated, minified, or stripped code does
- Diffing two versions of a closed artifact to explain what changed
- Any time the next step would be to act on the reconstruction in a way with real consequences — crafting a packet, flashing firmware, patching a binary, driving hardware, publishing a finding

**When NOT to use:**

- The target has authoritative documentation or a published spec — that's `source-driven-development`, not reverse engineering. Only fall back to this skill for the undocumented remainder.
- You have the actual source code and can just read it — that's ordinary code reading, not reconstruction from an opaque artifact.
- The target/tooling decision itself (which REA tool to call, how to open a binary) — see `reverse-engineer-anything`, which this skill treats as a toolbox, not a replacement.
- Trivial lookups where a wrong guess costs nothing (checking a well-known file magic number, a documented standard header).

**Authorization check, once, up front:** before investing in deep reconstruction — and always before Step 6 (GATE) — confirm the target is one you're authorized to analyze: something you own or licensed, an authorized pentest/CTF engagement, interoperability research, or defensive security work. This skill is about *accuracy*, not about deciding whether the work should happen; if authorization is unclear, stop and ask before continuing.

## Confidence Tiers

Every claim about the target carries one of three tiers. State the tier whenever you report a finding — to the user or to yourself in the ledger.

| Tier | Meaning | Example |
|---|---|---|
| **OBSERVED** | Directly read from the artifact or a capture — no interpretation layer beyond transcription | "Byte offset 0x14 is `0x4D5A` (`MZ`)." "The function at `0x401020` calls `memcpy` per the disassembly." "The capture shows a 4-byte big-endian length field preceding the payload in all 12 samples." |
| **INFERRED** | A conclusion drawn from OBSERVED evidence via reasoning — correct interpretation is likely but not independently confirmed | "This function appears to validate an HMAC before dispatch, based on the SHA-256 constant table and the branch structure." |
| **ASSUMED** | Filled in from naming, convention, domain knowledge, or "looks like X I've seen before" — not yet checked against the artifact at all | "This is probably a heartbeat packet because it's sent every 30 seconds, like most protocols do." |

The tier is not a formality — it changes what you're allowed to do with the claim:

- **ASSUMED** claims may not be reported as fact, and may not drive any action. Label them, or go gather evidence.
- **INFERRED** claims may be reported as findings, clearly labeled, but need a second independent evidence channel (Step 2) before they can drive an action with real consequences.
- **OBSERVED** claims still need a citation (the exact address/offset/capture ID), but don't need corroboration — the citation *is* the corroboration, because anyone can re-read the same bytes.

## The Process

Copy this checklist when applying the skill:

```
RE cycle:
- [ ] Step 1: HYPOTHESIZE — wrote the minimal falsifiable claim
- [ ] Step 2: CORROBORATE — gathered evidence from independent channels, cited exactly
- [ ] Step 3: CLASSIFY — tagged the claim OBSERVED / INFERRED / ASSUMED
- [ ] Step 4: DOUBT — adversarial pass against the evidence before promotion
- [ ] Step 5: LEDGER — recorded claim, tier, citations, contradictions, open questions
- [ ] Step 6: GATE — stopped for explicit authorization before any real-world action
```

### Step 1: HYPOTHESIZE — Name the falsifiable claim

Write the claim in one or two sentences, specific enough that evidence could contradict it:

```
HYPOTHESIS: Function sub_401020 validates an HMAC-SHA256 over the
            packet body before passing it to the dispatcher.
WHY THIS MATTERS: if wrong, a crafted packet that skips validation
                  would reach the dispatcher — this is the auth gate.
```

If you can't state it this precisely, you have a vibe, not a hypothesis — narrow it before gathering evidence. A hypothesis too broad to falsify ("this binary handles networking") can't be corroborated or doubted; decompose it into the smallest claim that matters for the question at hand.

### Step 2: CORROBORATE — Independent evidence channels

A single evidence channel tells you what one tool/lens produced — not what the artifact does. Decompilers guess at types and inlined control flow; a symbol table can be stripped or deliberately misleading; a single packet capture might be an edge case. Treat each of these as a **separate channel**, and prefer corroborating a claim across at least two before trusting it:

| Channel | What it gives you | What it can get wrong |
|---|---|---|
| Static disassembly/decompilation | Structure, control flow, constants, calls | Decompiler heuristics on types, inlining, or optimized code can misrepresent intent |
| Strings, symbols, xrefs | Naming hints, cross-references | Names can be stale, stripped, or adversarially misleading |
| Dynamic trace / instrumentation | What the code actually did on a given run | Only covers the paths exercised; a different input takes a different path |
| Protocol/file captures (multiple samples) | Observed structure across real instances | A pattern seen in 3 samples may not hold for the 4th |
| Differential comparison (versions, inputs, patched vs. unpatched) | Isolates what changed, which narrows cause | Unrelated changes between versions can be mistaken for the relevant one |

Cite evidence exactly, the way `source-driven-development` cites a doc URL — a reader must be able to re-check it:

```
BAD:  "It looks like this validates the signature."
GOOD: "sub_401020 (0x401020–0x4010B4) computes SHA-256 over
      bytes [buf+0x8, buf+0x8+len) and compares the result to
      buf[0x4:0x8] via memcmp at 0x401098 — confirmed against a
      dynamic trace of 5 captured packets (trace IDs t1–t5),
      all of which rejected when the comparison byte was flipped."
```

When using `reverse-engineer-anything` / REA tools, their returned Evidence IDs and graph references satisfy this citation requirement directly — carry them into the ledger rather than re-deriving them.

If a second channel contradicts the first, that contradiction is the finding — don't average the two into a comfortable middle guess. Surface it:

```
CONTRADICTION:
Static read suggests the length field is 2 bytes (struct layout
at offset 0x4). The dynamic trace shows a 4-byte value consumed
at that offset in all 8 captured packets.
→ Struct definition is likely wrong (padding/alignment, or a
  different struct version). Re-derive the layout from the trace,
  don't trust the static guess.
```

### Step 3: CLASSIFY — Tag the tier honestly

Apply the Confidence Tiers table above. Be suspicious of your own urge to round INFERRED up to OBSERVED just because the inference feels obvious — "obvious" is exactly the state of mind doubt-driven development exists to interrupt.

### Step 4: DOUBT — Adversarial pass before promotion

Before an INFERRED claim is promoted to drive an action, or reported to the user as a confirmed finding, subject it to the same adversarial mechanics as `doubt-driven-development` — but the ARTIFACT is the evidence, not code, and the question is alternative explanations, not code quality:

```
Adversarial review. Given only this evidence, find every alternative
explanation consistent with it that would make the HYPOTHESIS wrong.
Do NOT assume the hypothesis is correct. Look for:
- Another code path that produces the same observed bytes/behavior
- Edge cases, inputs, or states the evidence doesn't cover
- Whether the evidence channels are actually independent (same bug
  in the decompiler and the trace tooling would corroborate nothing)
- Anti-analysis behavior (the target detecting and altering behavior
  under instrumentation)

EVIDENCE: <citations gathered in Step 2>
HYPOTHESIS: <the claim from Step 1>

Find issues, or state explicitly that no alternative explanation
survives scrutiny.
```

If `doubt-driven-development` is available in session, use its fresh-context reviewer mechanics directly for this step rather than self-reviewing — a reviewer that only sees the evidence, not your reasoning, is exactly the colder read this step needs. Reconcile findings the same way: contract misread (the hypothesis was stated ambiguously — tighten it and re-loop), valid/actionable (gather more evidence or revise the hypothesis), valid trade-off (document and proceed), or noise (the reviewer lacked a channel you have — note whether that channel should have been included in EVIDENCE).

Skip this step only for OBSERVED claims that don't drive any action — a plain transcription of bytes doesn't need an adversary, it needs a citation.

### Step 5: LEDGER — Persist the conclusion, not the journey

RE sessions generate disassembly dumps, string lists, and capture logs that will either blow the context budget or vanish at compaction. Per `context-engineering`, extract the *decision* before the detail is dropped:

```
LEDGER ENTRY
Claim:        sub_401020 validates HMAC-SHA256 before dispatch
Tier:         INFERRED → promoted after Step 4 (no surviving
              alternative explanation)
Evidence:     static 0x401020-0x4010B4; dynamic traces t1-t5
              (REA evidence IDs ev_victory_07, ev_victory_08)
Contradicts:  none
Open question: untested with a packet shorter than the minimum
              observed length — behavior there is unconfirmed
```

Keep this ledger as a standing artifact (file, REA evidence bundle, or pinned message) that survives context compaction — not buried in conversation history. On a fresh session or after compaction, re-read the ledger before re-deriving anything; don't re-run analysis whose conclusion is already recorded, and don't trust a ledger entry whose "open question" line was never closed.

### Step 6: GATE — Stop before anything with real-world blast radius

Reconstructing understanding is reversible. Acting on it often isn't: crafting and sending a packet, flashing firmware, patching a live binary, replaying a captured transaction, running recovered credentials, or publishing an exploit. Before any such action:

1. Re-confirm authorization for *this specific action*, not just the investigation in general — understanding a system and being cleared to act on it are different approvals.
2. State the smallest possible real-world test that would confirm or falsify the hypothesis, and nothing larger.
3. Stop and get explicit user approval for that specific test before running it.

```
PROPOSED TEST: Send one crafted packet with the comparison byte
               flipped to a disposable test instance (not production),
               expecting rejection per the HYPOTHESIS.
→ Waiting for approval before sending anything.
```

This mirrors the "propose exactly ONE experiment and wait" pattern from quick vendor-binary RE passes, generalized to any target where a wrong action is expensive, destructive, or irreversible.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The symbol/string name says what it is" | Names are evidence, not proof. Stripped, stale, or adversarially misleading names are common — especially in anything built to resist inspection. |
| "I already decompiled it, that's confirmed" | One decompilation is one channel. Decompiler heuristics on types, inlining, and optimized control flow routinely produce plausible pseudocode for the wrong semantics. |
| "It's obviously doing X" | "Obviously" is the state right before a wrong inference hardens into a reported fact. That's exactly what Step 4 (DOUBT) exists to interrupt. |
| "I don't have time to corroborate everything" | Only claims that get reported as fact or drive an action need a second channel. Exploratory sketching can stay labeled INFERRED/ASSUMED while you work. |
| "Close enough / directionally right" | RE errors compound silently — an action built two layers on top of a wrong inference fails in a way that's hard to localize back to the original bad claim. |
| "I'll just try it and see what happens" | That's the Step 6 GATE being skipped. On hardware, production systems, or anything costly to undo, propose the smallest test and wait for approval instead. |
| "I'm confident, no need to doubt this one" | The artifact may have been specifically built to look plausible under casual inspection (optimization, obfuscation, anti-tamper). Confidence here is a weaker signal than almost anywhere else doubt-driven development applies. |
| "The ledger is overhead, I'll remember it" | Disassembly and capture context gets compacted or dropped; the conclusion doesn't survive unless it's written down separately. |

## Red Flags

- Reporting an INFERRED or ASSUMED claim to the user without its tier
- A claim backed by exactly one evidence channel being treated as settled
- Taking an action with real blast radius (flashing, exploiting, replaying, publishing, patching live) without an explicit Step 6 approval
- Treating a recovered name, string, or comment as authoritative intent rather than a hint to verify
- No ledger — findings exist only in conversation and disappear on compaction
- Skipping Step 4 (DOUBT) because the hypothesis "feels obvious"
- Reverse engineering a target that already has authoritative documentation — that's `source-driven-development`'s job
- Proceeding on an unclear authorization boundary instead of stopping to ask
- Averaging two contradicting evidence channels into a comfortable middle guess instead of surfacing the contradiction

## Interaction with Other Skills

- **`reverse-engineer-anything`**: supplies the tools (REA's static/dynamic analysis across native, managed, JS/Electron, Android, firmware, and browser targets). This skill supplies the discipline for interpreting what those tools return — carry REA's Evidence IDs directly into the Step 5 ledger rather than re-deriving citations.
- **`doubt-driven-development`**: Step 4 (DOUBT) is a specialized instance of its adversarial review, with ARTIFACT = gathered evidence and CONTRACT = the hypothesis, instead of code + contract. Reuse its fresh-context reviewer and cross-model escalation mechanics rather than reinventing them.
- **`source-driven-development`**: handles the parts of the target that *do* have authoritative documentation. Use this skill only for the undocumented remainder — if you find yourself reconstructing something with a published spec, switch skills.
- **`context-engineering`**: the Step 5 ledger is this skill's application of that skill's "compress before dropping" and "restartable session boundaries" guidance to RE's especially bulky, disposable artifacts.
- **`security-and-hardening`**: if reconstruction surfaces a vulnerability, hand off to that skill for responsible handling rather than defaulting to exploitation.
- **`debugging-and-error-recovery`**: once a hypothesis is confirmed and the next step is writing a fix, patch, or workaround, that's ordinary debugging — this skill's job (reconstructing the unknown) is done.

## Verification

After applying evidence-driven reverse engineering:

- [ ] Every claim reported to the user carries its tier (OBSERVED / INFERRED / ASSUMED)
- [ ] Every OBSERVED claim cites exact evidence (address, offset, instruction, capture/trace ID)
- [ ] Every claim promoted past ASSUMED has evidence from at least one channel, with claims driving an action corroborated across at least two independent channels
- [ ] Claims with real-world blast radius passed an adversarial DOUBT pass before being acted on or reported as confirmed
- [ ] Contradictions between evidence channels were surfaced explicitly, not averaged away
- [ ] A ledger exists mapping claim → tier → evidence → contradictions → open questions, and was written somewhere that survives context compaction
- [ ] No irreversible or system-touching action was taken without explicit, action-specific user authorization
- [ ] Any part of the target with authoritative documentation was handed to `source-driven-development` instead of reconstructed from scratch
