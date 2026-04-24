# Contributing

This guide aims to be **opinionated, compact, and citation-backed**. PRs that
sharpen any of those three properties are very welcome. PRs that broaden it
toward "everything you could ever want to know about .NET" are usually not —
the smaller it stays, the more it teaches.

## What works well as a PR

- A factual correction (with a link to the authoritative source).
- A new sub-section that fills a real hole, kept tight (under ~30 lines) and
  ending with a Sources block.
- An updated citation when an upstream doc moves or is superseded.
- A new anti-pattern in `monorepo-and-antipatterns.md` — please include the
  failure mode, the fix, and a real-world reference where possible.
- Bumping the "Target: .NET X / C# Y" header on a major release and reconciling
  inline guidance.

## What usually doesn't

- Tutorials. There are excellent ones on Microsoft Learn — link out instead.
- Tooling wars. We pick a default and link to alternatives in one line.
- Style nits — the markdown is intentionally a little informal.
- Net-new files without first opening an issue to scope the page.

## Style

- One sentence per line is fine — keeps diffs reviewable.
- Code blocks are minimal and runnable in isolation. Prefer real APIs over
  pseudo-code.
- End every page with a `## Sources` section listing the URLs you drew from.
- Don't editorialize about specific people or companies; critique the
  pattern, not the practitioner.

## Local checks

There's no build to run — just preview the markdown locally and make sure
links resolve. If you add a new top-level page, link it from `README.md`.

## Licensing

By contributing you agree that your contribution is licensed under the
project's MIT license (see [`LICENSE`](./LICENSE)).
