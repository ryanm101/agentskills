---
description: The same narrow single-symbol lookup that the re skill owns. Guards the "trivial lookups / already-bounded single-symbol pass" exclusion in evidence-driven-reverse-engineering's description, since a one-symbol decompile request must not pull in the heavier evidence-tier/ledger workflow.
expected_outcome: A quick decompile and control-flow summary of the one symbol; evidence-driven-reverse-engineering is never invoked.
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
---

Quick one: what does the symbol `crc32_table_init` do in this vendor `.so`? Just need the control flow, nothing fancier.
