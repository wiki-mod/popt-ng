# `calloc()` arguments transposed in `poptReadFile()`

**Fixed by:** [wiki-mod/popt-ng#NN](...) (this branch: `fix-poptreadfile-calloc-transposed-args`)
**Upstream location:** `src/poptconfig.c`, function `poptReadFile`, line 141
**Checked against upstream commit:** [`14c42b41`](https://github.com/rpm-software-management/popt/commit/14c42b415ba0c11640f7d4ee80920452d0287058) (`master`, checked 2026-08-20)
**Searched upstream issues/PRs for:** `calloc`, `poptconfig` -- no existing issue or PR found addressing this; the only calloc-adjacent PR found (#50, "Warnings") touches unrelated files (`popt.c`, `popthelp.c`, `poptint.c`, etc.), not `poptconfig.c`. Two recent CVE fixes in the same general area (CVE-2026-18739 in `poptStuffArgs()`, CVE-2026-18743 in `poptConfigFileToString()` in `src/poptparse.c`) are different functions in different files, unrelated to this finding.

## The problem

`poptReadFile()` allocates a buffer to hold a config file's contents with
`calloc(sizeof(*b), (size_t)nb + 1)`. `calloc`'s documented signature is
`calloc(nmemb, size)` -- number of elements first, size of each element
second. Here the arguments are transposed: `sizeof(*b)` (the element
size) is passed first, and the actual element count (`nb + 1`) second.

GCC 14 added a new `-Wcalloc-transposed-args` warning specifically to
catch this class of mistake, and this call trips it. Under
`-Werror` (as this fork's own build uses), it turns into a hard
compile failure on any toolchain with GCC 14+.

**Functionally harmless in this specific case, but still a real
inconsistency:** `b` is declared as `char *`, so `sizeof(*b)` is always
`1`. `calloc(1, nb+1)` and `calloc(nb+1, 1)` compute the identical total
allocation size (`nmemb * size` is commutative here, and a factor of `1`
can't introduce an overflow difference either way) -- there is no actual
memory-safety bug in this particular call. This is a portability/build
hygiene fix (keeps the code compiling clean under a newer, stricter
compiler and matches the project's own correct usage two lines away, see
below), not a security fix.

## Upstream code (unchanged as of the commit above, upstream)

```c
    if ((nb = lseek(fdno, 0, SEEK_END)) == (off_t)-1
     || (uintmax_t)nb >= SIZE_MAX
     || lseek(fdno, 0, SEEK_SET) == (off_t)-1
     || (b = calloc(sizeof(*b), (size_t)nb + 1)) == NULL
     || read(fdno, (char *)b, (size_t)nb) != (ssize_t)nb)
```

Notably, the same file already gets this right elsewhere
(`poptDupArgv`, around line 103):

```c
	if (avp && (*avp = calloc((size_t)(1 + 1), sizeof (**avp))) != NULL)
```

-- number of elements first, size second. The transposed call in
`poptReadFile` is an isolated inconsistency, not the file's general
style.

## Fixed code (changed code as of the commit from this fork)

```c
    if ((nb = lseek(fdno, 0, SEEK_END)) == (off_t)-1
     || (uintmax_t)nb >= SIZE_MAX
     || lseek(fdno, 0, SEEK_SET) == (off_t)-1
     || (b = calloc((size_t)nb + 1, sizeof(*b))) == NULL
     || read(fdno, (char *)b, (size_t)nb) != (ssize_t)nb)
```

## Empirical verification

All commands below were run on a real remote host (never local), against
a fresh, direct `git clone` of this fork.

**Step 1 -- isolated before/after repro of this exact fix**, inside the
mandated `ghcr.io/wiki-mod/distcc-ng-buildtools:latest` container (Debian
13/trixie, GCC 14.2.0), against the identical source vendored into
`wiki-mod/distcc-ng`'s `popt/` fallback tree (same upstream 1.19 origin):

Before the fix:

```console
$ ./configure --without-system-popt PYTHON="$(which python3)"
$ make popt/poptconfig.o
...
popt/poptconfig.c: In function 'poptReadFile':
popt/poptconfig.c:141:27: error: 'calloc' sizes specified with 'sizeof'
  in the earlier argument and not in the later argument
  [-Werror=calloc-transposed-args]
make: *** [Makefile:501: popt/poptconfig.o] Error 1
$ echo $?
2
```

After the fix (transposing the two arguments):

```console
$ ./configure --without-system-popt PYTHON="$(which python3)"
$ make popt/poptconfig.o
gcc [...] -o popt/poptconfig.o -c popt/poptconfig.c
$ echo $?
0
```

**Step 2 -- this repository's own real CI recipe (`ci/Dockerfile`), with
the fix applied, on a current, supported Fedora release** (this PR also
bumps the CI image itself, see below):

```console
$ docker build -f ci/Dockerfile -t popt-verify .
$ docker run --rm popt-verify
Test project /srv/popt/build
    Start 1: test1_build
1/6 Test #1: test1_build ......................   Passed    0.98 sec
    Start 2: test2_build
2/6 Test #2: test2_build ......................   Passed    0.16 sec
    Start 3: test3_build
3/6 Test #3: test3_build ......................   Passed    0.14 sec
    Start 4: tdict_build
4/6 Test #4: tdict_build ......................   Passed    0.15 sec
    Start 5: tstuff_build
5/6 Test #5: tstuff_build .....................   Passed    0.14 sec
    Start 6: testit
6/6 Test #6: testit ...........................   Passed    0.56 sec

100% tests passed, 0 tests failed out of 6
```

**Companion CI fix, same PR:** `ci/Dockerfile` was still pinned to
`fedora:39`, which reached end-of-life on 2024-11-26 (this project's own
history shows it was bumped 36 -> 39 back in 2024-03-04 for the same
staleness reason -- `1a1076c8`, "Bump CI to Fedora 39, 36 is getting a bit
long in the tooth"). Bumped to `fedora:44` (current supported release).
This surfaced the `-Werror=calloc-transposed-args` failure above in the
first place: Fedora 39's GCC 13 doesn't have that check yet, so the
transposed-args bug was silently compiling clean there.

**Also added in this PR:** a `.gitattributes` (`* text=auto eol=lf`,
excluding `*.pdf`) -- this repository had none, so a contributor checking
out on Windows with the (very common) default `core.autocrlf=true` gets
every text file re-written with CRLF line endings on checkout, which then
breaks `#!/bin/sh` shebang lines and file-comparison tests the moment that
checkout is used anywhere outside Windows itself. Confirmed this isn't a
pre-existing content problem in the repository itself first: a fresh,
direct `git clone` of this fork on a real Linux host, followed by `git
status`/`git diff --stat` after running a CRLF normalizer over the whole
tree, showed zero actual changes -- the real, committed content is
already consistently LF. `.gitattributes` makes that guaranteed by policy
rather than by accident.

## Upstream status

Still present in upstream's live source as of the commit checked above.
Not reported upstream yet.
