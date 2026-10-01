# Fork notes: upload-integrity fork of reva

This fork carries fixes for upload-integrity gaps until upstream ships equivalent
changes. Goal: an upload either lands byte-for-byte intact, or fails loudly.

- Upstream: https://github.com/opencloud-eu/reva
- Base: `11d87fb6b985` (the commit pinned by opencloud's `go.mod`)
- Branch layout:
  - `fix/*`: one branch per fix, each based directly on the base commit and free of
    fork-only files, so it can be sent upstream as-is.
  - `integrity`: base + this file + all fixes. opencloud's `replace` directive points here.
- When rebasing onto a new upstream release, drop every commit upstream has
  superseded and re-run the full torture test (`devtools/integrity-test/` in the
  opencloud fork) before deploying.

## Running the tests

The posix driver needs Linux (xattrs, inotify), and `-race` needs cgo, so run tests in a
Linux container, not on Windows. `tests/helpers.TempDir` puts test data in `<repo>/tmp`,
which must live on a Linux filesystem: on a Windows bind mount, symlinks and xattrs
misbehave and about 34 decomposedfs/posix tree and recycle specs fail even on an
unmodified upstream checkout.

```sh
docker run --rm -v "$PWD:/src" -v reva-tmp:/src/tmp -w /src golang:1.26 \
  sh -c 'apt-get update -qq && apt-get install -qq -y inotify-tools >/dev/null &&
         go test -race ./pkg/rhttp/datatx/... ./pkg/storage/...'
```

## Divergences from upstream

### 1. `fix(tus): serialize concurrent PATCH requests per upload`

Branch `fix/tus-serialize-patch`.

- `pkg/rhttp/datatx/manager/tus/tus.go`: registers tusd's `memorylocker` on the store
  composer (if the storage didn't already register a locker).
- `pkg/storage/pkg/decomposedfs/upload/upload.go`:
  - `WriteChunk` returns tusd's 409 `ERR_MISMATCHED_OFFSET` if the staging file size
    differs from the offset tusd passed in.
  - `FinishUploadDecomposed` rejects an upload whose staging file size differs from
    the declared size. The session is cleaned up, and the error is
    `errtypes.ChecksumMismatch` (the stored bytes don't match what the client
    declared). Until fix 4, tusd reports it as a 500. With fix 4 it becomes 460.
- Tests: `pkg/rhttp/datatx/manager/tus/tus_concurrency_test.go` (real tusd handler +
  decomposedfs over HTTP: a stalled PATCH overlapped by a HEAD + PATCH retry, and a
  plain sequential resume); `upload_test.go` (`TestWriteChunkRejectsStaleOffset`,
  `TestFinishUploadRejectsSizeMismatch`).
- Before the fix, the overlap test stores `src[0:16K] + src[16K:32K] + src[16K:32K]`
  (duplicated bytes) and the finalized blob's sha256 differs from the source.
- Behaviour notes:
  - With the locker, a new request for an upload interrupts the request holding the
    lock. The interrupted request gets tusd's 400 `ERR_UPLOAD_INTERRUPTED`; the new
    one re-reads the offset under the lock, so a stale `Upload-Offset` gets 409.
  - `AcquireLockTimeout` stays at tusd's default of 20s. A request that is blocked
    in a disk write (not a body read) can't be interrupted, so a retry waits up to
    20s and then gets a 500 `ERR_LOCK_TIMEOUT`. Clients retry that.
  - `memorylocker` only serializes requests within one storage-users process. For
    multiple replicas on shared storage, switch to `filelocker` rooted in the uploads
    directory.
  - The legacy copy `pkg/storage/utils/decomposedfs` (used only by the `ocis` and
    `s3ng` drivers) has the same `WriteChunk`/`FinishUploadDecomposed` code and is
    deliberately not changed (out of scope: posix driver only).

Upstream issue draft:

> **tus: concurrent PATCH requests to one upload can store duplicated bytes**
>
> The tus datatx manager (`pkg/rhttp/datatx/manager/tus/tus.go`) builds a tusd
> `StoreComposer` without a locker. tusd documents a locker as required to prevent
> corruption from concurrent access to one upload. decomposedfs' `WriteChunk`
> ignores the offset and opens the staging file with `O_APPEND`, so two requests
> that both pass tusd's offset check both append.
>
> Trigger: a client retries after a timeout (the desktop client sends HEAD + PATCH
> right away on chunk timeout) while the original request is still being written
> server-side, e.g. behind a buffering proxy. Uploads without a checksum (all web
> uploads today) are then finalized with duplicated bytes and no error.
>
> Reproduction: `tus_concurrency_test.go` in the linked PR. It drives the real tusd
> handler over HTTP against decomposedfs and fails on current main.

Upstream PR draft: title as the commit subject; body = the commit message, plus
"Fixes #<issue>". Attach the test output before and after the change.

## Upstream issues in dependencies (not patched here)

- **tusd v2.10.0/v2.10.1 data race** (`pkg/handler/context.go:65` vs
  `unrouted_handler.go` `writeChunk`): the goroutine that closes the body on
  `ErrUploadInterrupted` reads `ctx.body` without synchronization while `writeChunk`
  assigns it. If a lock-release request arrives after a PATCH took the lock but before
  it created its body reader, the race detector fires and the interruption is lost:
  the first request keeps the lock and completes, and the waiting request waits for
  it (or times out after 20s). This doesn't corrupt data, but it trips `go test -race`.
  The fork's tests avoid that window. Worth reporting to github.com/tus/tusd.

## Notes for the owner

- Nothing re-verifies stored checksums at rest. Run the storage on ZFS or btrfs with
  scheduled scrubs.
