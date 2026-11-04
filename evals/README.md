# Plugin evals

Checks that `re` and `evidence-driven-reverse-engineering` fire on the prompts they're meant for, stay quiet on prompts that belong to the other one (or to no RE skill at all), and change the reply the way each skill promises.

Uses Claude Code's built-in `claude plugin eval` (2.1.269+), which loads this whole plugin and scores each case with and without it. One case = one directory under `evals/plugin/` holding `prompt.md` (frontmatter: turn/tool limits; body: the prompt, sent verbatim) and one grader per file under `graders/`.

## Running

```bash
claude plugin eval . --no-publish                                  # every case, both arms, 3 runs each
claude plugin eval . --case re-fires-on-vendor-symbol --runs 1      # one cheap iteration on one case
```

Reports land in `evals/plugin/results/<timestamp>/` (gitignored). Every run and every `llm` grader is a real model call on your account — run on demand, never in CI.

## Cases

| Case | Checks |
|---|---|
| `re-fires-on-vendor-symbol` | A narrow "find this symbol, decompile it, summarize control flow" request fires `re` and the reply follows its stop-and-propose-one-experiment shape |
| `re-stays-quiet-on-refactor` | An ordinary code refactor, unrelated to RE, does not fire `re` |
| `evidence-driven-reverse-engineering-fires-on-protocol-reconstruction` | Reconstructing an undocumented protocol from captures, headed toward a real replay, fires `evidence-driven-reverse-engineering` and the reply uses confidence tiers and gates the replay on approval |
| `evidence-driven-reverse-engineering-stays-quiet-on-symbol-lookup` | The same narrow single-symbol lookup that `re` owns does not also pull in the heavier skill |

Skill invocation is stochastic — read the report's fire rate and reply quality across runs, not a single pass/fail.
