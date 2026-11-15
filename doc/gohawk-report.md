# gohawk analysis — November 15, 2026

## Scope and reproduction

- Target: `golang.org/x/pkgsite`, checkout `cd189d9b19a3d5f9ede283f6d980ea0ce2c54522`,
  including the pre-existing uncommitted changes in `cmd/worker/main.go`,
  `internal/queue/{queue.go,gcpqueue/queue.go,gcpqueue/queue_test.go}`, and
  `internal/worker/{server.go,server_test.go}`. Those changes were not modified.
- Tool: gohawk v0.8.1, built and run with Go 1.27.1 on linux/amd64.
- Pattern: `./...`, matching 94 packages in this environment.
- gohawk explicitly excludes `_test.go` files. Ordinary Go files in test-helper
  and test-tool packages are included. This is static analysis, not a test run.
- The working-tree diff before adding this report had SHA-256
  `4dfdcbb9af19d8da99efb4ba81ad85f68b6c78459a9596b5849940af00788c6d`
  (`git diff --binary | sha256sum`).

```sh
GOBIN="$HOME/go/bin" go install github.com/kojah/gohawk@v0.8.1
"$HOME/go/bin/gohawk" -json ./...
"$HOME/go/bin/gohawk" -enable-all -json ./...
```

Both runs completed with exit status 3 (findings), with byte-identical JSON:
four diagnostics across three packages. There were no reported build/analysis
errors. The default run took 2m49s, including dependency downloads; the all-checks
run took 1m45s with a warm build cache. Enabling all checks, including experimental
process ownership, added no findings.

The accompanying `gohawk-results.json` preserves the shared output, with only the
absolute checkout prefix removed from source locations.

## Findings and manual triage

| Source location | Check | Assessment |
| --- | --- | --- |
| `internal/database/database.go:410` | `deferinloop/cleanup-lifetime` | Benign bounded retention |
| `internal/database/database.go:417` | `deferinloop/cleanup-lifetime` | Benign bounded retention |
| `internal/testing/testhelper/testhelper.go:140` | `resourcelifetime/missing-release` | Real file-descriptor leak on an error path |
| `tests/api/main.go:304` | `resourcelifetime/missing-release` | Real file-descriptor leak on error paths |

### Two database warnings: cleanup is deliberately function-scoped

`(*DB).bulkInsert` batches inserts. The first full batch prepares a statement
only when `stmt == nil` and defers its close at line 410. Later full batches reuse
that statement. A final partial batch may prepare one additional statement and
defer its close at line 417.

Although both defers are syntactically inside the loop, each can execute at most
once per call. Thus there are at most two prepared statements, both closed when
the function returns. These are not unbounded accumulation or missing-cleanup
bugs; no fix appears warranted merely to satisfy these diagnostics. Moving the
first close to the end of each iteration would break statement reuse.

### Test helper: file is not closed if writing fails

In `writeTestDirectory`, `os.Create` succeeds at line 140, but a
`file.WriteString(contents)` error at lines 144–145 returns before reaching
`file.Close()` at line 147. This is a genuine descriptor leak on write failure.

Suggested fix: move the work for one file into a helper that installs cleanup
immediately after successful creation, or explicitly close on write failure
while preserving the original error. Avoid simply adding a defer inside the
existing loop, which would retain all successfully written files until return.

### API test tool: output and archive-entry handles can escape cleanup

In `writeZip`, `os.OpenFile` succeeds at line 304. If opening the zip entry fails
at lines 308–310, the output file is never closed; this is the error path
identified by gohawk.

Manual inspection also shows that an `io.Copy` error at lines 312–313 skips
closing both `outFile` and `rc`, and an `outFile.Close` error at lines 315–316
skips closing `rc`. These are additional paths around the same code, not
additional gohawk diagnostics.

Suggested fix: extract per-entry copying into a helper, install cleanup for each
handle immediately after acquisition, and preserve useful copy/close errors.

## Interpretation

Four warnings reduce to two actionable cleanup sites, both in testing support
code, plus two apparently benign database warnings. There were no diagnostics
from cancellation ownership, concurrent capture, goroutine ownership, lock
ordering, producer lifecycle, or experimental process ownership. This does not
prove those properties: gohawk's supported patterns and uncertainty handling
limit coverage.

No application code was changed or findings suppressed as part of this run.
