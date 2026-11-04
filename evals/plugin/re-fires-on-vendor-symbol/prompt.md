---
description: A narrow request to find and decompile one symbol in a vendor binary and summarize its control flow. The re skill should fire and the reply should follow its stop-and-propose-one-experiment shape, not dive into a full evidence ledger.
expected_outcome: strings/nm commands, a Ghidra decompile, a control-flow summary, and a single proposed runtime experiment held for approval; re is invoked.
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
---

We have a vendor-supplied binary at /opt/vendor/libradio.so with no source or docs. I just need a quick pass on the symbol `radio_set_power` — find it, decompile it, and tell me what it does to the hardware registers before I go anywhere near the actual radio.
