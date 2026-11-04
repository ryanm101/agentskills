---
name: re
description: Safe reverse-engineering pass on a vendor binary symbol
---
1. Run `strings -a $BINARY | rg -i '<topic>'` and `nm -D $BINARY | rg <symbol>`.
2. Decompile the matching function with Ghidra headless (`analyzeHeadless ./ghidra_proj tmp -import $BINARY -postScript Decompile.java`).
3. Summarize the control flow and any ioctl/fourcc constants found.
4. STOP. Propose exactly ONE runtime experiment and wait for my approval before touching hardware.
