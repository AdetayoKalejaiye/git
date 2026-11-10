# Data Structures and Algorithms Map

This repository is the Git source tree. It is primarily a C codebase with
small support libraries in `xdiff/`, `reftable/`, `ewah/`, and the hash
implementations. This document maps the structures and algorithms that form
the main execution paths; it is not an inventory of every helper type.

## 1. High-level data flow

```text
command (`builtin/*.c`)
    |
    +-- repository/config/environment setup
    |       (`setup.c`, `repository.c`, `config.c`, `environment.c`)
    |
    +-- object names and refs
    |       (`object-name.c`, `refs/`, `reftable/`)
    |
    +-- object database
    |       loose objects or packfiles
    |       (`object-file.c`, `odb/`, `packfile.c`)
    |
    +-- object graph
    |       commits -> parents, trees -> entries, trees -> blobs
    |       (`commit.c`, `tree.c`, `blob.c`)
    |
    +-- index/worktree state
    |       (`read-cache.c`, `unpack-trees.c`, `dir.c`)
    |
    +-- revision walk / diff / merge
            (`revision.c`, `commit-reach.c`, `diff*.c`, `merge*.c`, `xdiff/`)
```

The central identifier is `struct object_id`, defined through `hash.h`. It
allows the object model to support SHA-1 and SHA-256 repositories without
making callers depend on the hash size.

## 2. Core content-addressed object model

| Structure | Meaning | Main files |
|---|---|---|
| `struct object` | Common header for all parsed Git objects: object ID, type, flags, and repository ownership. | `object.h`, `object.c` |
| `struct commit` | Commit object plus parsed parent list, author/committer data, timestamp, and commit graph metadata. | `commit.h`, `commit.c` |
| `struct tree` | Directory snapshot identified by a tree object ID. | `tree.h`, `tree.c`, `tree-walk.c` |
| `struct blob` | File contents identified by a blob object ID. | `blob.h`, `blob.c` |
| `struct tag` | Annotated tag object pointing to another object. | `tag.h`, `tag.c` |
| `struct commit_list` | Singly linked lists of commits used by reachability, merge-base, and history code. | `commit.h`, `commit-reach.c` |
| `struct object_list` / `struct object_array` | Linked and growable-array collections of arbitrary Git objects. | `object.h`, `object.c` |
| `struct parsed_object_pool` | Repository-local object lookup table and allocation pools for parsed objects. | `object.h`, `object.c`, `alloc.c` |

Object parsing is lazy. Lookup first finds or creates an object by ID; parsing
then reads and validates the object only when a caller needs its type-specific
fields. `object-file.c` and `odb/` abstract the source of the bytes.

Trees are stored as sorted name/mode/object-ID entries. `tree-walk.c` exposes
parallel tree descriptors so commands can compare or traverse directory
snapshots without materializing every path at once.

### Blob objects in detail

A blob is Git's immutable byte container for file content. It does not store a
pathname, permissions, timestamps, or a link to a commit; those relationships
are supplied by tree entries and commits. Two files with identical bytes have
the same blob object ID, even when they have different names or appear in
different commits.

The blob lifecycle is:

1. A producer reads bytes from the worktree, stdin, or another object.
2. Git constructs the canonical object header (`"blob <length>\\0"`).
3. The header and content are hashed with the repository's configured hash
   algorithm to produce an `object_id`.
4. The object is written as a loose compressed object or included in a pack.
5. A tree entry associates the blob ID with a pathname and file mode.
6. Readers resolve the ID through the object database and return a buffer or
   streaming handle.

`struct blob` is intentionally small because the content remains in the
object database rather than in the parsed object structure. `lookup_blob()`
and `parse_blob_buffer()` in `blob.c` connect the generic object cache to the
blob type. `read_loose_object()`, the object-database read interfaces, and the
streaming APIs in `odb/streaming.c` handle the actual bytes.

Blob-related algorithms and policies include:

- **Content addressing:** the object ID changes if any byte changes, making
  the ID both a lookup key and an integrity check.
- **Canonical hashing:** callers hash the object type and byte length along
  with content, so a blob cannot collide with identical raw bytes interpreted
  as a tree or commit.
- **Streaming:** large blobs can be copied, filtered, or hashed incrementally
  instead of being fully resident in memory.
- **Delta reconstruction:** packed blobs may be stored as a delta against a
  similar blob; reads recursively apply base data and delta instructions.
- **Binary/text classification:** diff code samples or examines blob content
  to decide whether to use textual line diff, binary reporting, or a driver.
- **Promisor access:** in a partial clone, a missing blob can trigger a fetch
  from a promisor remote, subject to the caller's missing-object policy.

Blob contents are consumed by `cat-file`, checkout, archive, diff, merge,
filter, clean/smudge, and pretty-printing paths. The most useful entry points
are `builtin/hash-object.c`, `builtin/cat-file.c`, `object-file.c`,
`odb/streaming.c`, `diff.c`, and `convert.c`.

## 3. Object database and storage indexes

### Loose objects

Loose objects are read and written through the object database interfaces in
`odb/`, `object-file.c`, `loose.c`, and `object-store-ll.h`. Content is
compressed and addressed by its object ID. Streaming interfaces in
`odb/streaming.c` avoid loading large blobs into memory.

The loose-object path has three important operations:

- **Lookup:** derive the fan-out path from the hexadecimal object ID and open
  the object file.
- **Decode:** inflate the zlib payload, parse the type/size header, and verify
  that the decoded size and requested type agree.
- **Write:** create the object in a temporary location, flush and close it,
  then atomically rename it into the object store. This prevents readers from
  observing a partially written object.

The object database (`struct object_database`) owns one or more object sources.
Sources can be searched in order, and alternates/promisor sources can supply
objects that are not present in the primary repository.

### Packfiles

| Structure | Role | Main files |
|---|---|---|
| `struct packed_git` | Open packfile metadata, index data, windows, and object counts. | `packfile.h`, `packfile.c` |
| `struct pack_entry` | A located object inside a pack, including offset and pack owner. | `packfile.h` |
| `struct pack_window` | Memory-mapped/windowed access to pack bytes. | `packfile.h`, `packfile.c` |
| `struct object_entry` | Mutable object record used while constructing a pack. | `pack-objects.h`, `pack-objects.c` |
| `struct packing_data` | Object table, delta candidates, and packing state. | `pack-objects.h`, `pack-objects.c` |

Pack lookup uses the pack index (fan-out table plus sorted object IDs) to find
an object, then resolves the object at its offset. Delta-compressed objects
are reconstructed by following a base-object chain. Pack creation uses
object ordering and delta heuristics to reduce size; see `pack-heuristics.adoc`
and `pack-objects.c`.

Pack indexes provide two levels of lookup: a 256-entry fan-out table narrows
the range of object IDs, then binary search locates the exact ID in the
sorted name table. The index supplies the corresponding pack offset and,
where available, CRC information. Large offsets use an auxiliary 64-bit
offset table.

Pack object headers encode the object type and compressed size using
continuation bits. A non-delta object can be inflated directly. An
`OFS_DELTA` stores a negative offset to its base; a `REF_DELTA` stores the
base object ID. Delta data contains a base size, result size, and copy/insert
instructions. Git validates both sizes while applying the delta and rejects
invalid chains.

Pack construction has separate phases: enumerate reachable objects, assign
object entries, search for delta bases, order entries, write compressed data,
and write the index. `struct object_entry` records flags, size, hash-table
links, and delta relationships needed across those phases.

### Multi-pack index and bitmaps

`midx.h` and `midx.c` provide a single lookup index across multiple packs.
Reachability bitmaps use compressed bitsets to represent sets of commits and
objects; the format and traversal rules are documented in
`Documentation/technical/bitmap-format.adoc` and implemented in
`pack-bitmap.c` and `ewah/`.

The multi-pack index avoids opening every pack index for each object lookup.
It assigns a stable pack-position coordinate to objects, while the bitmap
layer uses those coordinates to answer “which objects are reachable?” without
enumerating every object. Bitmap operations are especially important for
clone, fetch, repack, and connectivity-check workloads.

## 4. The index and working tree

The index is Git’s staging area and a cache of filesystem/object metadata.

| Structure | Role | Main files |
|---|---|---|
| `struct cache_entry` | One indexed path: mode, object ID, stat data, stage, flags, and flexible-array path name. | `read-cache-ll.h`, `read-cache.c` |
| `struct index_state` | The in-memory index: sorted cache entries, cache tree, name/dir hash maps, split-index and sparse-index state. | `read-cache-ll.h`, `read-cache.c` |
| `struct cache_tree` | Cached tree object IDs for index subtrees, allowing `write-tree` to reuse unchanged directory results. | `cache-tree.h`, `cache-tree.c` |
| `struct untracked_cache` | Directory-level cache for untracked-file discovery. | `dir.h`, `dir.c`, `untracked-cache.c` |
| `struct split_index` | Shared-index representation that stores unchanged entries separately from the active index. | `split-index.h`, `split-index.c` |

Index entries are sorted by pathname, with conflict stages allowing multiple
versions of one path during a merge. `index_name_pos()` uses binary search;
`name-hash.c` builds hash maps for fast exact path and directory lookups.
Sparse-index support represents entire out-of-scope directories with a
directory entry instead of expanding all files; see
`Documentation/technical/sparse-index.adoc`.

Each `cache_entry` combines four kinds of state:

- the pathname and file mode used to construct a tree;
- the blob object ID staged for that pathname;
- stat data used to avoid rereading unchanged worktree files;
- flags for conflict stages, skip-worktree, intent-to-add, fsmonitor state,
  and transient update/removal decisions.

The index is not itself a tree object and is not content-addressed. It is a
mutable operational cache whose checksum protects the on-disk file. Its
sorted cache array is the canonical path order; the name and directory
hashmaps are derived accelerators and can be rebuilt.

The common checkout/update path is:

1. Read or refresh `index_state` from the index file.
2. Compare index entries with trees and/or filesystem stat data.
3. Use `unpack-trees.c` and checkout code to populate or update the worktree.
4. Write a new index under a lock using `lockfile.c` and `read-cache.c`.

`cache-tree.c` groups adjacent sorted index paths into directory nodes. A
node is valid when all entries below it are unchanged; `write-tree` can then
reuse its tree object ID. If an entry changes, validity is invalidated up the
ancestor chain and only affected subtrees are rewritten.

During a merge, stage 1 is the merge base, stage 2 is “ours,” and stage 3 is
“theirs.” A resolved path is collapsed to stage 0. `unpack-trees.c`,
`read-tree.c`, and merge strategies use these stages to represent unresolved
conflicts without losing either side's object ID.

## 5. References and reflogs

References map names such as `refs/heads/main` to object IDs. The reference
backend API is in `refs.h` and `refs.c`, with filesystem storage in `refs/files`
and the reftable backend in `reftable/`.

Reftable is an append-friendly, block-oriented table format:

| Structure | Role | Main files |
|---|---|---|
| `struct reftable_ref_record` | A ref name and its value or deletion/update state. | `reftable/reftable-record.h` |
| `struct reftable_log_record` | A reflog entry. | `reftable/reftable-record.h` |
| `struct reftable_table` | One immutable table with block/index offsets. | `reftable/reftable-table.h` |
| `struct reftable_merged_table` | A read view over multiple tables ordered by recency. | `reftable/merged.h` |
| `struct reftable_stack` | The table stack and compaction/write coordination. | `reftable/stack.h`, `reftable/stack.c` |

Reftable lookups use sorted blocks and indexes; newer tables shadow older
records. Compaction merges tables while preserving the newest visible value.
The design is described in `Documentation/technical/reftable.adoc`.

Reference transactions collect multiple updates before committing them. The
backend verifies expected old object IDs, locks or stages the relevant files,
writes reflog entries as required, and publishes the new values atomically.
This prevents a concurrent writer from silently overwriting a ref that moved
after the caller read it.

Reflogs are append-oriented histories of ref updates. They are separate from
the commit graph: a reflog can retain the only convenient name for an
otherwise unreachable commit, while garbage collection later uses reachability
and expiration policy to decide what can be pruned.

## 6. Commit graph and revision walking

`struct rev_info` in `revision.h` is the state machine for commands that
select, filter, order, and display revisions. It contains pending objects,
commit work queues, pathspec/filter settings, and traversal flags.

The walk uses:

- `struct commit_list` for simple linked-list work.
- `struct prio_queue` (`prio-queue.h`) for date-order, topological, or custom
  priority ordering. With no comparator it acts as a LIFO stack.
- Object flags such as `SEEN`, `UNINTERESTING`, `TREESAME`, and `BOUNDARY` to
  avoid revisiting commits and to carry traversal state.
- Commit slabs (`commit-slab.h`) for sparse per-commit metadata without
  modifying the commit structure.

Revision setup resolves command-line names into pending objects, expands
reference patterns, applies exclusions, and marks boundary/uninteresting
objects. Iteration then repeatedly selects the next commit, parses its
parents when needed, applies path and message predicates, and inserts eligible
parents into the work queue. The flags on each object make this multi-stage
process incremental and prevent duplicate work.

Important algorithms and entry points:

| Algorithm | Purpose | Implementation |
|---|---|---|
| Reachability walk | Visit parents from one or more tips with pruning and filters. | `revision.c`, `list-objects.c` |
| Topological/date ordering | Emit commits while respecting parent-before-child constraints and ordering policy. | `revision.c`, `graph.c`, `prio-queue.c` |
| Merge-base computation | Find common ancestors and remove stale/redundant candidates. | `commit-reach.c` |
| Ahead/behind counts | Count commits reachable from one side but not the other. | `commit-reach.c` |
| Commit-graph acceleration | Store generation numbers and optional Bloom filters to reduce graph/object work. | `commit-graph.c`, `commit-graph.h`, `bloom.c` |
| Shallow/promisor boundaries | Stop or defer traversal at shallow and remotely promised objects. | `shallow.c`, `promisor-remote.c`, `list-objects-filter*.c` |

The commit-graph format stores commits in graph order, parent edges, and
generation data. Generation numbers provide lower bounds that let reachability
and merge-base walks stop exploring commits that cannot improve the answer.
Changed-path Bloom filters can skip commits whose trees cannot contain a
requested path. See `Documentation/technical/commit-graph.adoc`.

Merge-base computation marks ancestors from each side, prioritizes newer
generation/date values, identifies common ancestors, and removes common
ancestors that are themselves reachable from better candidates. The result is
not necessarily a single commit: criss-cross histories can have multiple
merge bases.

Topological ordering is stronger than timestamp ordering. A child is held
until the required parent relationships have been accounted for, while
date-order uses commit dates to choose among currently available candidates.
`graph.c` maintains display metadata such as lanes and merges for graph
rendering; it is separate from the underlying reachability walk.

## 7. Diff and merge algorithms

### Tree and file comparison

`diff.c` builds `struct diff_filespec` values for old and new files and pairs
them in `struct diff_filepair`. `diffcore_std()` applies transformations such
as break-rewrite, rename/copy detection, pickaxe filtering, and ordering.
The internal structures are declared in `diffcore.h`; implementations include
`diffcore-rename.c`, `diffcore-break.c`, `diffcore-pickaxe.c`, and
`diffcore-order.c`.

Rename/copy detection compares deleted and added files using content
similarity scores. It can break a rewrite into delete/add components and then
match likely pairs subject to configured limits.

The diff pipeline is deliberately staged:

1. Produce file pairs from two trees, two indexes, or an index and worktree.
2. Normalize or populate file data only when an operation requires content.
3. Run diffcore transformations such as break detection, rename/copy
   detection, pickaxe search, and path/order filtering.
4. Render the resulting pairs as patches, statistics, raw records, or
   machine-readable formats.

`struct diff_filespec` delays blob loading through its object ID and records
whether data is borrowed, heap-allocated, or memory-mapped. This is important
for large repositories: metadata-only operations need not read every blob.
Rename similarity is based on shared content after the selected text/binary
handling and is bounded by rename limits to avoid quadratic comparisons.

### Line diff

`xdiff/` implements the line-oriented difference engine. The default engine in
`xdiff/xdiffi.c` is based on Eugene Myers’ `O(ND)` shortest-edit-script
algorithm, with heuristics for expensive cases. Git also exposes:

- patience diff (`XDF_PATIENCE_DIFF`);
- histogram diff (`XDF_HISTOGRAM_DIFF`);
- minimal mode (`XDF_NEED_MINIMAL`);
- whitespace and indent heuristics.

The public xdiff interface is in `xdiff/xdiff.h`; Git’s adapter and output
formatting are in `xdiff-interface.c` and `diff.c`.

Myers diff searches edit-graph diagonals for a shortest path, where deletions
and insertions are edits and equal runs are “snakes.” The implementation
stores furthest-reaching positions per diagonal and splits the problem
recursively. Minimal mode continues searching for an optimal edit script;
heuristic mode can stop at a good-enough split for pathological inputs.
Patience diff first uses unique lines as anchors and then applies a fallback
diff to the regions between anchors. Histogram diff chooses low-frequency
lines as anchors and is often more stable for source code with repeated
boilerplate.

### Three-way merge

Text merging uses xdiff merge routines and merge drivers in `merge-ll.c`.
Tree-level strategies are implemented in `merge-recursive.c` and `merge-ort.c`.
They combine merge bases, tree traversal, rename detection, directory rename
detection, and index/worktree updates. The ort strategy keeps structured
conflict and path state to reduce repeated work compared with the older
recursive implementation.

At a high level, a tree merge recursively compares base/ours/theirs entries:

- unchanged-on-both-sides entries are reused;
- changes on only one side are taken directly;
- identical changes on both sides are coalesced;
- file/file, directory/file, mode, and rename conflicts are classified;
- conflicting blobs are passed to a merge driver or xdiff;
- the result is written as a merged tree or as staged index entries.

Directory rename detection derives directory-level evidence from individual
renames and maps paths into the renamed directory when the evidence is strong
enough. The result is advisory and must still handle add/add and modify/delete
conflicts explicitly.

## 8. Reusable collections and algorithms

| Structure/utility | Algorithm or representation | Files |
|---|---|---|
| `struct hashmap` | Separate-chaining hash table with caller-provided equality function; embedded `hashmap_entry` avoids wrapper allocations. | `hashmap.h`, `hashmap.c` |
| `struct oidset` | Hash-backed set specialized for object IDs. | `oidset.h`, `oidset.c` |
| `struct oid_array` | Growable array of object IDs, optionally sorted/deduplicated. | `oid-array.h`, `oid-array.c` |
| `struct string_list` | Growable string/value list with optional sorting and ownership rules. | `string-list.h`, `string-list.c` |
| `struct strmap` / `struct strset` | String-keyed maps and sets built on hash maps. | `strmap.h`, `strmap.c` |
| `struct strbuf` | Geometrically growing, NUL-terminated byte buffer. | `strbuf.h`, `strbuf.c` |
| `struct mem_pool` | Arena-style allocation for many short-lived related objects. | `mem-pool.h`, `mem-pool.c` |
| Commit slabs | Array-like sparse storage indexed by commit allocation number. | `commit-slab.h`, `commit-slab-impl.h` |
| EWAH bitmap | Word-aligned hybrid run-length compressed bitmap. | `ewah/`, `ewah/ewah_bitmap.c` |
| `struct prio_queue` | Binary heap when ordered; dynamic array/LIFO behavior otherwise. | `prio-queue.h`, `prio-queue.c` |
| `struct list_head` | Intrusive doubly linked list used by low-level subsystems. | `list.h`, `list.c` |
| `struct cbtree` | Cache-friendly ordered tree used for selected lookup workloads. | `cbtree.h`, `cbtree.c` |

Most collections make ownership explicit through `*_init`, `*_clear`, and
`*_release` functions. Many objects are arena-allocated and released in bulk
at repository or operation boundaries.

The main complexity tradeoffs are intentional:

- hashmaps provide expected constant-time lookup but require a stable
  comparison function and rehashing as the table grows;
- sorted arrays provide cache locality and binary search but make insertion
  more expensive;
- linked lists make insertion cheap but have poor locality and linear search;
- arenas reduce allocation overhead but defer individual object reclamation;
- compressed bitmaps trade update cost and random mutability for very fast
  set unions/intersections and compact reachability storage.

## 9. Path matching, traversal, and filtering

Path-oriented commands combine:

- `struct pathspec` (`pathspec.h`) for literal, wildcard, attribute, and
  magic-prefix path expressions;
- directory traversal and ignore matching in `dir.c`/`dir.h`;
- tree traversal in `tree-walk.c`;
- object filtering in `list-objects-filter.c` and
  `list-objects-filter-options.c`;
- pathspec/Bloom-filter pruning during revision walks.

The result is a layered pruning strategy: reject paths early from the
worktree/index, avoid descending into irrelevant tree entries, and avoid
opening commits whose changed-path metadata proves they cannot match.

Pathspec matching distinguishes literal components, wildcard components,
recursive `**` patterns, and magic modifiers such as case-insensitive or
attribute-aware matching. Directory traversal can prune a subtree when its
prefix cannot match any pathspec, while ignore/exclude rules are evaluated
with directory context.

Object filters used by partial clone and fetch describe what to omit or
include: blobs by size, sparse paths, or tree depth are common examples.
Filters are applied while enumerating reachable objects, and missing promised
objects remain valid until a later operation explicitly requires them.

## 10. Where to start reading

For a command-to-storage trace, start with:

1. `builtin/log.c` -> `revision.c` -> `commit-reach.c` for history queries.
2. `builtin/write-tree.c` -> `cache-tree.c` -> `tree.c` for index-to-tree
   conversion.
3. `builtin/cat-file.c` -> `object-file.c` -> `odb/`/`packfile.c` for object
   reads.
4. `builtin/diff.c` -> `diff.c` -> `diffcore-*.c` -> `xdiff/` for diffs.
5. `builtin/merge.c` -> `merge-ort.c` -> `merge-ll.c` for merges.
6. `builtin/update-ref.c` -> `refs.c` -> `refs/files` or `reftable/` for ref
   updates.

The corresponding format and design notes are concentrated in
`Documentation/technical/`, especially `commit-graph.adoc`,
`bitmap-format.adoc`, `reftable.adoc`, `sparse-index.adoc`, and
`pack-heuristics.adoc`.

For blob-specific traces, follow:

1. `builtin/hash-object.c` -> `index-pack`/`object-file.c` to see content
   hashing and object creation.
2. `builtin/cat-file.c` -> object-database reads or
   `odb_stream_blob_to_fd()` to see object reads and streaming.
3. `builtin/add.c` -> `convert.c` -> `read-cache.c` to see worktree bytes
   become staged blob IDs.
4. `builtin/show.c`/`builtin/archive.c` -> tree walking -> blob reads to see
   committed file content materialized.
5. `diff.c` -> `diff_populate_filespec()` -> `odb/`/`packfile.c` to see lazy
   blob loading during comparison.

For performance and correctness work, also inspect the relevant tests under
`t/`, especially tests whose names begin with `t14` (object database and
packfiles), `t20` (index and worktree), `t40` (diff/merge), and `t55`
(partial clone and object filtering).
