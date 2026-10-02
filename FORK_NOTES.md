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

### 2. `fix(posix): restore revisions atomically and guard upload rollback`

Branch `fix/atomic-restore-revision`.

- `pkg/storage/fs/posix/tree/revisions.go` `RestoreRevision`: copies the revision into
  a temp file in `<space>/.oc-tmp`, fsyncs it, copies the target's `user.oc.*` xattrs,
  mode and owner onto it (best effort for the owner: a failed chown is logged, not
  fatal), applies the revision's checksum, blobid, blobsize and type attributes and
  the mtime, then `rename()`s it over the target. The mtime is then set once more
  through the metadata backend so its cache picks up the new attributes. The "current"
  copy for `EnableFSRevisions` is unchanged.
- `pkg/storage/pkg/decomposedfs/upload/upload.go` `Cleanup` (the `versionID` branch):
  takes the node's metadata lock and re-reads the status through the metadata
  backend, bypassing the node's own attribute cache. It only restores when the node is
  still `processing:<this session>`. Otherwise it logs, keeps the node content, and
  leaves the revision as an ordinary version. The revision is deleted only after a
  successful restore, as before.
- The lock is released before the rest of `Cleanup` runs. `UnmarkProcessing` locks
  again through a different node object, and flock locks on separate file
  descriptors would self-deadlock.
- Tests: `pkg/storage/fs/posix/tree/revisions_test.go` (restore keeps node metadata;
  a failed copy leaves the target untouched; concurrent readers only ever see the old
  or the new content); `upload_async_test.go` "two uploads overwrite an existing file
  in parallel".
- Before the fix, a failed copy truncates the target to 0 bytes, 3831 of 3841
  concurrent reads saw a partial file, and the aborted older upload rolled the file
  back from 20 to 10 bytes.
- Not changed: downloads still don't take the node lock. With an atomic rename they
  don't need it to get consistent bytes (an open fd keeps the old inode). The
  decomposed driver's `RestoreRevision` only rewrites xattrs and wasn't touched.

Upstream issue draft:

> **posix: RestoreRevision overwrites the live file in place; aborted uploads can roll
> back newer content**
>
> 1. `posix/tree/revisions.go` `RestoreRevision` truncates the target and `io.Copy`s
>    the revision into it. Readers see truncated or mixed data; a failed copy leaves
>    the file empty while its xattrs describe the old content.
> 2. `decomposedfs/upload` `Cleanup` restores the session's `versionID` revision
>    without the node lock and without checking that the session still owns the node.
>    Unlike the branch below it, it doesn't compare `ProcessingID`. If postprocessing of
>    an older upload fails after a newer upload has finished, the newer content is
>    replaced by the version from before the older upload.
>
> Reproduction: `revisions_test.go` and the new `upload_async_test.go` case in the
> linked PR, both failing on current main.

### 3. `fix(posix): write the fs-revisions copy from the stored file`

Branch `fix/fs-revisions-copy`.

- `pkg/storage/fs/posix/blobstore/blobstore.go` `Upload`: the copy target (used only
  when `STORAGE_USERS_POSIX_ENABLE_FS_REVISIONS=true`) is now read from
  `n.InternalPath()` instead of the upload source, which the default rename path has
  already moved. The copy is fsynced and closed with error checks.
  `canUseRenameForUpload` is an `atomic.Bool`, read once per upload.
- Tests: `blobstore_copytarget_test.go` (copy target on both the rename and copy
  paths; `Tree.WriteBlob` with `enable_fs_revisions`; concurrent cross-device uploads
  under `-race`).
- Before the fix, `Upload` with a copy target on the rename path failed with "could
  not open source file", so `Tree.WriteBlob` failed and the upload would stay in
  processing. `-race` reported the `canUseRenameForUpload` write/read race.
- Keep `STORAGE_USERS_POSIX_ENABLE_FS_REVISIONS` **off** in production regardless.

Upstream issue draft:

> **posix: enable_fs_revisions makes every upload fail at finalize**
>
> `posix/blobstore` `Upload` renames the upload source into place and then
> `os.Open(source)`s it again to write the copy target that `Tree.WriteBlob` passes
> when `EnableFSRevisions` is set. The open fails and `WriteBlob` returns an error
> after the blob was stored, so finalization fails. Also, `canUseRenameForUpload` is
> a plain bool written by concurrent uploads (data race under `-race`).
> Reproduction: `blobstore_copytarget_test.go` in the linked PR.

### 4. `fix(tus): return 460 for checksum mismatches instead of 500`

Branch `fix/tus-error-status`. The desktop half lives in the desktop fork.

- `pkg/storage/pkg/decomposedfs/upload/upload.go` `FinishUpload`:
  `errtypes.ChecksumMismatch` → tusd `ERR_CHECKSUM_MISMATCH` **460** (tus checksum
  extension status), `errtypes.BadRequest` → **400**. Previously both were a generic
  500.
- The size mismatch from fix 1 is an `errtypes.ChecksumMismatch` too, so it also maps
  to **460** (decision: the stored bytes don't match what the client declared, and the
  session is gone, so the client must start over, same as a checksum mismatch).
- 460 is relayed unchanged by the datagateway and by ocdav's creation-with-upload path.
  The plain-PUT datatx paths keep their existing 419 for checksum mismatches.
- Client impact: the desktop client classifies 460 as `NormalError` (retried). The
  desktop fork additionally clears its resume info on 460. The web client already
  refused to retry 5xx responses (its own `onShouldRetry`), and tus-js-client does not
  retry 4xx other than 409/423, so for web uploads only the reported status changes.
- Tests: `upload_status_test.go` (unit, mapping) and `tus_status_test.go` (real tusd
  handler over HTTP: wrong checksum → 460, correct checksum → 204). Before the fix the
  HTTP test got 500.

Upstream issue draft:

> **tus: checksum mismatch at the end of an upload is reported as 500**
>
> `decomposedfs/upload` `FinishUpload` maps only `AlreadyExists` and `Aborted` to tusd
> errors. A `ChecksumMismatch` (and `BadRequest`) falls through and tusd sends a 500,
> although the upload session has already been deleted. Clients treat 500 as
> transient and keep retrying a dead upload URL. The tus checksum extension defines
> 460 Checksum Mismatch for this case.

## Found by the torture test, not fixed (outside the handoff's scope)

### Restoring a version with the same mtime as the current file loses that version

Reproduced on upstream (`opencloud-eu/opencloud` `a1982c6908`, unmodified reva) and on
the fork alike, so it isn't caused by fix 2.

1. Upload content A, then content B to the same file, both with the same mtime (clients
   that preserve mtimes do this: the desktop client, the web client's `lastModified`,
   `rsync -t`, two copies of the same file).
2. Restore the version (A): `COPY /remote.php/dav/meta/<fileid>/v/<id>.REV.<mtime>`.
3. The server answers 204, but the file still contains B. The version list now shows
   `<id>.REV.<mtime>.1`, and downloading it fails with 500, so A is unreachable.

Cause (by reading): `Decomposedfs.RestoreRevision` first calls `CreateRevision` for the
current content, keyed by its mtime. With equal mtimes that key is the revision being
restored, so `CreateRevision`'s collision handling renames A to `.REV.<mtime>.1` and
writes B into `.REV.<mtime>`. The restore then copies that (B) back, and the
`.1` revision's metadata no longer matches.

Upstream issue draft:

> **Restoring a version with the same mtime as the current file keeps the current
> content and makes the version unreadable**
>
> Steps: upload A and then B to one file with identical mtimes (X-OC-Mtime / tus
> `mtime`), restore the only version. Expected: content A. Actual: 204, content stays
> B, the version is renamed to `.REV.<mtime>.1` and GET on it returns 500.
> `RestoreRevision` creates a revision of the current node with the same key as the
> revision being restored (`CreateRevision` → `os.IsExist` → rename).

### 0-byte files have no stored checksums

`oc:checksums` is empty for 0-byte files (TUS and PUT), while every other size gets
SHA1/MD5/ADLER32. The content is correct (empty). Clients that compare checksums can't
verify empty files. This was noted, not changed.

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
