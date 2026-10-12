# Contributing

Thanks for helping keep this guide accurate and useful. It is a living reference for production AI systems and interview prep, so contributions grounded in real-world practice are especially welcome.

## What to contribute

High-value additions include:

- **Interview questions** with staff-level depth and worked answers.
- **Production failure modes** you have hit, plus how you diagnosed and fixed them.
- **Model benchmarks** and pricing updates for confirmed, publicly released models.
- **Corrections** to anything outdated, ambiguous, or wrong.
- **New patterns** (RAG, agents, MCP, eval pipelines, multi-tenant isolation) with concrete tradeoffs.

## How to propose changes

1. Fork the repo and create a branch from the latest `main`.
2. Make your change in focused commits with logical boundaries.
3. Open a pull request describing what changed and why. Link a source for any new model, price, or benchmark claim.

For small fixes (typos, broken links, stale numbers), a direct PR is perfect. For larger additions such as a new chapter or case study, open an issue first so we can align on scope.

## Style conventions

- **No em dashes.** Use commas, parentheses, or a rewrite instead. This applies to prose, commit messages, and code comments.
- **Confirmed-real models only.** Reference models that have actually shipped publicly. Do not invent model names, versions, prices, or benchmark numbers. When you cite pricing or capabilities, link the source.
- **Say who measured it.** Label vendor-reported numbers as vendor-reported. Cite a benchmark score with its version, reasoning effort, harness, and runner (public leaderboard or vendor), because the same model can differ by several points across those. Describe announced, preview, and beta features as exactly that, not as shipped.
- **Date time-sensitive facts in the sentence** ("released September 22, 2026", "as of October 2026") instead of adding "Last verified" stamps, which belong only in the model taxonomy, pricing, and job-market chapters.
- **American spelling and straight quotes.** Use "-ize" endings and spellings such as "behavior" and "labeling", and use `"` and `'` rather than curly quotes.
- **Conventional commits.** Use messages like `docs: fix broken links in case studies` or `feat(retrieval): add ColBERT reranking notes`, scoped to one logical change each.
- **Match the surrounding voice.** Opinionated, concrete, and tradeoff-aware. Prefer specific numbers and named tools over vendor-neutral hedging.
- **Keep links relative** and verify they resolve before opening a PR.

## Quality bar

- Be accurate. If you are unsure, say so or leave it out.
- Be concrete. Name the model, the metric, the failure mode, the cost.
- Be current. This guide tracks what ships, so flag anything that has aged out.

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE).
