# Support Upstream

This directory is a passive, read-only offering to upstream
[`rpm-software-management/popt`](https://github.com/rpm-software-management/popt).
It exists on the same premise as `wiki-mod/distcc-ng`'s own
`support-upstream/` directory: a bug found and fixed here may still be a
real bug in upstream's own, independently-maintained source, whether or
not upstream ever adopts anything from this fork.

Every entry here is a self-contained Markdown writeup that:

- names the exact file and line in upstream's current source where the
  problem lives (pinned to the upstream commit it was checked against,
  since upstream keeps moving independently of this fork),
- shows the actual "before" code and this fork's "after" fix,
- explains *why* it's a real bug, not a stylistic preference,
- and includes real empirical verification evidence where the finding
  warrants it (built and tested on a real host, not just "the diff looks
  right").

## Required template

Every entry documenting a bug still live in upstream (the common case)
should use these section headers, in this order:

```markdown
# <short, specific title>

**Fixed by:** [wiki-mod/popt-ng#NN](...)
**Upstream location:** `src/foo.c`, function `bar`
**Checked against upstream commit:** [`<sha>`](<commit url>) (`master`, checked <date>)
**Searched upstream issues/PRs for:** `<terms>` -- <what was/wasn't found>

## The problem

## Upstream code (unchanged as of the commit above, upstream)

## Fixed code (changed code as of the commit from this fork)

## Empirical verification
```

`## Empirical verification` may be omitted only when the finding is
trivial enough that no build/test evidence is needed to see it's real.

## Index

| # | Fixed By | Title | Upstream Status |
|---|---|---|---|
| 1 | (this PR) | [`calloc()` arguments transposed in `poptReadFile()`](poptreadfile-calloc-transposed-args.md) | Still present in upstream's live source (`master`), unreported |
