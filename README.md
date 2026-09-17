# hgit

A Git implementation written **100% in H#** — the H# analogue of Rust's
[`git2`](https://docs.rs/git2) crate: a from-scratch, in-language Git
object model and porcelain, not a wrapper around the real `libgit2` C
library or the `git` binary.

```
use "hlib -> hgit" from "hgit"

let repo = hgit::repository::init(".", false)
match repo is
    hgit::error::GitResult::Ok(r)  => write("initialized at " + r.path())
    hgit::error::GitResult::Err(e) => write("error: " + e.to_string())
end
```

## What's in here

hgit reimplements the parts of Git that make up ~everything a normal
workflow touches, structured the same way `git2` splits things into
plumbing types and porcelain convenience calls:

| Module | git2 equivalent | Covers |
|---|---|---|
| `oid` | `Oid` | Content-addressed 40-hex-char object ids, SHA-1 hashing |
| `error` | `Error` / `Result<T, E>` | `GitError` + the generic `GitResult<T>` every fallible call returns |
| `object` | `Odb` | Loose-object storage: hash / write / read, shared by blob/tree/commit/tag |
| `blob` | `Blob` | File content |
| `tree` | `Tree` / `TreeBuilder` | Directory snapshots, entry lookup, a mutable builder |
| `commit` | `Commit` | Tree + parents + author/committer + message |
| `tag` | `Tag` | Annotated tag objects |
| `signature` | `Signature` | Name/email/timestamp/offset, Git's exact signature-line format |
| `refs` | `Reference` | `refs/heads/*`, `refs/tags/*`, `HEAD`, symbolic + direct refs |
| `index` | `Index` | The staging area: add/remove paths, `write_tree`/`read_tree` |
| `repository` | `Repository` | `init` / `open` / `discover`, the entry point tying paths together |
| `branch` | `Branches` + tag half of `Repository` | Create/list/delete/rename branches, lightweight + annotated tags |
| `checkout` | `build::CheckoutBuilder` | Materialize a tree/commit/branch into the working directory |
| `revwalk` | `Revwalk` | Commit history traversal, `git log`-order, ancestor checks |
| `diff` | `Diff` | LCS line diffs between blobs, added/deleted/modified between trees |
| `status` | `Statuses` | Three-way HEAD ↔ index ↔ working-directory comparison |
| `reset` | `Repository::reset()` | Soft / mixed / hard reset |
| `ignore` | `git_ignore_*` | Minimal `.gitignore` pattern matching |
| `gitconfig` | `Config` | Reads/writes `.git/config`'s `[section "sub"]` / `key = value` format |
| `clone` | `Repository::clone()` | **Local-path** clone (see Known limitations) |
| `porcelain` | — | `commit_all`, `log`, `merge_fast_forward`, `is_clean`: the everyday one-call wrappers |

`lib.h#` re-exports every module from one place, so a consumer only
needs one `use`/`mod` line (see "Using hgit in your own project" below).

## Quickstart

```
use "hlib -> hgit" from "hgit"
use "std -> fs" from "fs"

fn main() is
    match hgit::repository::init("/tmp/demo", false) is
        hgit::error::GitResult::Err(e) => write(e.to_string())
        hgit::error::GitResult::Ok(repo) => is
            fs::write("/tmp/demo/hello.txt", "hi\n")

            let me = hgit::signature::Signature::now("Ada", "ada@example.com")
            let commit_id = hgit::porcelain::commit_all(repo, "first commit", me)

            match commit_id is
                hgit::error::GitResult::Ok(id) => write("committed " + id.short())
                hgit::error::GitResult::Err(e) => write(e.to_string())
            end
        end
    end
end
```

More complete, runnable-shaped examples live in `examples/`:

- `examples/init_and_commit.h#` — init, stage, commit, log
- `examples/branch_and_diff.h#` — branch, fast-forward merge, tree diff

## Design notes — read before assuming byte-for-byte `git` compatibility

hgit is **content-addressed the same way real Git is**: an object's id
is a SHA-1 hash of its content, objects are immutable, deduplicated,
and stored under `.git/objects/<aa>/<...>`; refs are files under
`.git/refs/...`; the index is a staging area between the working
directory and the next commit; history is a DAG of commits each
pointing at a tree and zero-or-more parents. A repository `hgit`
creates, commits into, branches, and diffs behaves exactly like Git
conceptually.

Three low-level encoding details are deliberately simplified relative
to the real `.git` on-disk format, because H# v0.9's stdlib is
string-oriented (there's no ergonomic, tested way yet to splice raw
non-UTF8 bytes into a text buffer the way Git's NUL-terminated headers
and raw 20-byte sha1 tree entries require):

1. **Loose-object header terminator is `\n`, not `\x00`.** A real
   loose object is `"<type> <len>\x00<payload>"`; hgit writes
   `"<type> <len>\n<payload>"`. See `object.h#`'s header comment.
2. **Tree entries reference children by 40-char hex oid, not Git's raw
   20-byte binary sha1.** See `tree.h#`'s header comment.
3. **The index (`.git/index`) is one plain-text line per staged file**
   (`<mode> <oid-hex> <path>`) instead of Git's packed binary v2/v3
   format with its SHA-1 checksum footer. See `index.h#`'s header
   comment.

Because of these, an id `hgit` computes will **not** match the id the
real `git` binary computes for equivalent content, and a repository
hgit creates is not readable by the real `git` CLI (and vice versa).
hgit is internally consistent — the same input always hashes to the
same id, and a repository round-trips perfectly through hgit itself —
but it's a from-scratch, Git-*shaped* version-control library
implemented natively in H#, not a `.git`-directory-reading drop-in
replacement for `git2`/`libgit2`. If a future H# stdlib release adds
ergonomic raw-byte buffer support, these three spots are exactly where
a byte-for-byte-compatible mode would slot in — nothing else in the
library's structure would need to change.

Loose objects are also round-tripped through `std -> archive`'s
`gzip_compress`/`gzip_decompress` — the slot real Git fills with raw
zlib deflate. That pair is a documented identity passthrough in the
current H# interpreter release (see `std/archive.h#`), so objects are
presently stored uncompressed on disk; every call site in `object.h#`
is already wired for the day that changes, with no changes needed
anywhere else in the library.

## Known limitations

Deliberately out of scope for this first pass — each is either a large
project on its own (packfiles, a real merge algorithm, the smart-HTTP
protocol) or blocked on stdlib features this H# release doesn't expose
yet (raw byte buffers, `chmod`):

- **No packfiles / no `packed-refs`.** Every object is a loose object;
  every ref is its own file. Fine for the repo sizes a library like
  this is aimed at; not what you'd want for a multi-gigabyte monorepo.
- **No network transport.** `clone.h#` clones from a local path only —
  no smart-HTTP/SSH client. See its header comment.
- **No three-way content merge.** `porcelain::merge_fast_forward` only
  fast-forwards; a real merge (or a conflicted one) returns
  `ErrorKind::Conflict` rather than attempting one. Fast-forward covers
  the "I branched, made a few commits, and nothing else touched the
  base branch meanwhile" case; anything that actually diverged needs a
  human (or a future contribution) to resolve.
- **File modes are recorded but not applied to disk**, and symlinks
  are materialized as plain text files containing the link target —
  `std -> fs` has no `chmod`/symlink-creation exposed yet. See
  `checkout.h#`'s header comment. The object model itself is correct
  regardless (a tree entry's mode is stored and round-trips fine);
  only turning that mode into real filesystem permissions during
  checkout is deferred.
- **`.gitignore` support is minimal**: exact paths, directory
  prefixes, and single-`*` glob segments. No `**`, no `!`-negation, no
  nested per-directory `.gitignore` files. See `ignore.h#`.
- **Diff output is a readable unified-style rendering, not a `git
  apply`-able patch** — no `@@` hunk headers, no context-line
  windowing, no rename detection.

None of these change the object model or the public API shape — they
narrow what a handful of functions currently *do*, not what they mean.

## Using hgit in your own project

As a `bytes` package dependency (once published as a `.hlib` — see
H#'s `config/HLIB_FORMAT.md`):

```
[deps]
-> hgit => hlib
```

```
use "hlib -> hgit" from "hgit"
```

While developing locally (or if you'd rather vendor the source
directly instead of depending on a built `.hlib`), copy `src/*.h#`
into your own project and include the top module the same way this
repo's own tests do:

```
mod lib   ;; if you keep hgit's src/ files under, say, src/hgit/
;; or, one `mod` per file you actually need:
mod oid
mod repository
```

## Running the tests

```
bytes test
```

Tests live in `src/*_test.h#` (next to the modules they exercise, not
in `tests/` — see `tests/README.md` for why) and cover:

- `oid_test.h#` — hashing, hex parsing, short ids, equality
- `object_test.h#` — blob/tree/commit round-trips, dedup, tree hash
  stability, wrong-type-lookup errors
- `repository_workflow_test.h#` — the end-to-end path: init, stage,
  commit, branch, checkout, log, status, reset --hard, tree diff

## Project layout

```
hgit/
├── Bytes.hk                       package manifest (emit = lib)
├── README.md                      this file
├── src/
│   ├── lib.h#                     re-exports every module
│   ├── error.h#  oid.h#  signature.h#
│   ├── object.h#  blob.h#  tree.h#  commit.h#  tag.h#
│   ├── refs.h#  index.h#  repository.h#
│   ├── checkout.h#  branch.h#  reset.h#
│   ├── revwalk.h#  diff.h#  status.h#
│   ├── ignore.h#  gitconfig.h#  clone.h#  porcelain.h#
│   └── *_test.h#                  unit + integration tests (see above)
├── tests/
│   └── README.md                  explains why tests live in src/
└── examples/
    ├── init_and_commit.h#
    └── branch_and_diff.h#
```

## License

MIT.
