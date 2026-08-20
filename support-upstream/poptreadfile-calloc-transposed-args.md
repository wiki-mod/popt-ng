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

Verified on a real remote host (not local), inside the mandated
`ghcr.io/wiki-mod/distcc-ng-buildtools:latest` container (Debian 13/trixie,
GCC 14.2.0), against the identical source vendored into
`wiki-mod/distcc-ng`'s `popt/` fallback tree (same upstream 1.19 origin,
identical `poptconfig.c` content at the time of this check -- this
specific verification was run against that vendored copy, not an
independent build of this repository's own CMake/autotools build system):

- **Before the fix:** `./configure --without-system-popt && make
  popt/poptconfig.o` fails with `error: 'calloc' sizes specified with
  'sizeof' in the earlier argument and not in the later argument
  [-Werror=calloc-transposed-args]`, exit code 2.
- **After the fix** (transposing the two arguments): the identical build
  command compiles `poptconfig.o` cleanly, exit code 0.
- **No regression:** a full `./configure --without-system-popt && make &&
  make check` run with the fix applied completes the build and the vast
  majority of the test suite (one unrelated, environment-specific test
  failure was observed and traced to the test container's own user setup,
  not this change).

## Upstream status

Still present in upstream's live source as of the commit checked above.
Not reported upstream yet.
