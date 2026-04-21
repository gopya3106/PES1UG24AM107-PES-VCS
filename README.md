 PES-VCS LAB REPORT

NAME: NAKKA GOPYA

SRN: PES1UG24AM107

**Platform:** Ubuntu 22.04

## Phase 1: Object Storage Foundation

**Filesystem Concepts:** Content-addressable storage, directory sharding, atomic writes, hashing for integrity

**Files:** `pes.h` (read), `object.c` (implement `object_write` and `object_read`)

### What to Implement

Open `object.c`. Two functions are marked `// TODO`:

1. **`object_write`** — Stores data in the object store.
   - Prepends a type header (`"blob <size>\0"`, `"tree <size>\0"`, or `"commit <size>\0"`)
   - Computes SHA-256 of the full object (header + data)
   - Writes atomically using the temp-file-then-rename pattern
   - Shards into subdirectories by first 2 hex chars of hash

2. **`object_read`** — Retrieves and verifies data from the object store.
   - Reads the file, parses the header to extract type and size
   - **Verifies integrity** by recomputing the hash and comparing to the filename
   - Returns the data portion (after the `\0`)

Read the detailed step-by-step comments in `object.c` before starting.

### Testing

```bash
make test_objects
./test_objects
```

The test program verifies:
- Blob storage and retrieval (write, read back, compare)
- Deduplication (same content → same hash → stored once)
- Integrity checking (detects corrupted objects)

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing.
<img width="940" height="215" alt="image" src="https://github.com/user-attachments/assets/6092ca5d-1da1-4d69-adaf-f52596248845" />

**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure.
<img width="940" height="102" alt="image" src="https://github.com/user-attachments/assets/78bbbc3d-4cbc-4853-bbbc-f427776877e2" />

---

## Phase 2: Tree Objects

**Filesystem Concepts:** Directory representation, recursive structures, file modes and permissions

**Files:** `tree.h` (read), `tree.c` (implement all TODO functions)

### What to Implement

Open `tree.c`. Implement the function marked `// TODO`:

1. **`tree_from_index`** — Builds a tree hierarchy from the index.
   - Handles nested paths: `"src/main.c"` must create a `src` subtree
   - This is what `pes commit` uses to create the snapshot
   - Writes all tree objects to the object store and returns the root hash

### Testing

```bash
make test_tree
./test_tree
```

The test program verifies:
- Serialize → parse roundtrip preserves entries, modes, and hashes
- Deterministic serialization (same entries in any order → identical output)

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing.
<img width="940" height="201" alt="image" src="https://github.com/user-attachments/assets/e7e8747b-8a98-4fb7-9590-d8487f393e92" />

**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format.
<img width="940" height="85" alt="image" src="https://github.com/user-attachments/assets/91d0b9a0-73c1-4563-9edc-e4e8d6cc42d2" />

---

## Phase 3: The Index (Staging Area)

**Filesystem Concepts:** File format design, atomic writes, change detection using metadata

**Files:** `index.h` (read), `index.c` (implement all TODO functions)

### What to Implement

Open `index.c`. Three functions are marked `// TODO`:

1. **`index_load`** — Reads the text-based `.pes/index` file into an `Index` struct.
   - If the file doesn't exist, initializes an empty index (this is not an error)
   - Parses each line: `<mode> <hash-hex> <mtime> <size> <path>`

2. **`index_save`** — Writes the index atomically (temp file + rename).
   - Sorts entries by path before writing
   - Uses `fsync()` on the temp file before renaming

3. **`index_add`** — Stages a file: reads it, writes blob to object store, updates index entry.
   - Use the provided `index_find` to check for an existing entry

`index_find` , `index_status` and `index_remove` are already implemented for you — read them to understand the index data structure before starting.

#### Expected Output of `pes status`

```
Staged changes:
  staged:     hello.txt
  staged:     src/main.c

Unstaged changes:
  modified:   README.md
  deleted:    old_file.txt

Untracked files:
  untracked:  notes.txt
```

If a section has no entries, print the header followed by `(nothing to show)`.

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index    # Human-readable text format
```

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output.
<img width="986" height="672" alt="image" src="https://github.com/user-attachments/assets/a6651d90-15c7-4632-817d-ac40a6671f5a" />

**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries.
<img width="1125" height="56" alt="image" src="https://github.com/user-attachments/assets/50782ccd-5e74-4f28-9684-34fd53032866" />

---

## Phase 4: Commits and History

**Filesystem Concepts:** Linked structures on disk, reference files, atomic pointer updates

**Files:** `commit.h` (read), `commit.c` (implement all TODO functions)

### What to Implement

Open `commit.c`. One function is marked `// TODO`:

1. **`commit_create`** — The main commit function:
   - Builds a tree from the index using `tree_from_index()` (**not** from the working directory — commits snapshot the staged state)
   - Reads current HEAD as the parent (may not exist for first commit)
   - Gets the author string from `pes_author()` (defined in `pes.h`)
   - Writes the commit object, then updates HEAD

`commit_parse`, `commit_serialize`, `commit_walk`, `head_read`, and `head_update` are already implemented — read them to understand the commit format before writing `commit_create`.

The commit text format is specified in the comment at the top of `commit.c`.

### Testing

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

You can also run the full integration test:

```bash
make test-integration
```

**📸 Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages.
<img width="1911" height="823" alt="image" src="https://github.com/user-attachments/assets/37c1e175-116d-4832-87f3-26d0270ba258" />

**📸 Screenshot 4B:** `find .pes -type f | sort` showing object store growth after three commits.
<img width="1078" height="288" alt="image" src="https://github.com/user-attachments/assets/8d6bb2f9-336e-412f-ab2f-e200254cc0e3" />

**📸 Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain.
<img width="999" height="71" alt="image" src="https://github.com/user-attachments/assets/ba0dba95-db59-4fef-a7ff-01f17c7f9d93" />

---

## Phase 5 & 6: Analysis-Only Questions

**Q5.1:** — How would you implement pes checkout <branch>?
Files that must change in .pes/:

.pes/HEAD — rewrite the line from ref: refs/heads/<current> to ref: refs/heads/<target>.
If the target branch is new, create .pes/refs/heads/<target> pointing to the desired commit hash.
Working-directory update:

Read the target branch tip from .pes/refs/heads/<branch>.
object_read the root tree of that commit.
Recursively walk the tree: for blob entries write (or overwrite) the file in the working directory; for tree entries (mode 040000) create directories.
Delete any working-directory file that is present in the current HEAD's tree but absent from the target tree.
What makes this complex:

Dirty-file conflicts. If a tracked file has local modifications and the target branch has a different version of that file, checkout must refuse or the user's edits are destroyed silently.
Untracked-file collisions. If an untracked file sits at a path that the target tree would create, checkout must refuse to avoid clobbering it.
Partial failure. A crash halfway through leaves a mixed working directory. A production implementation writes a lock or in-progress marker to allow recovery.
Three-way diff. Deciding which files to touch requires comparing the current-HEAD tree, the target-HEAD tree, and the index all at once.

**Q5.2:— Detecting dirty working-directory conflicts
Using only the index and the object store:

Load the current index (the staged snapshot matching HEAD after the last commit).
For each entry, stat() the working-directory file. If st_mtime or st_size differs from the stored values, re-hash the file (SHA-256) and compare to the stored hash. A mismatch means the file is dirty.
For each path in the target branch's tree: if that path is in the index and is dirty (step 2), refuse checkout for that path.
For untracked files: if a path in the target tree does not appear in the index but already exists on disk, refuse checkout to avoid overwriting it.
The mtime/size check is a fast first pass; the full hash re-read is the authoritative check used only when metadata differs.

**Q5.3:Detached HEAD and recovery
When HEAD holds a raw commit hash instead of ref: refs/heads/<branch>, head_update writes new hashes directly into HEAD. Commits still chain correctly via parent pointers, but no branch file is updated. Once you check out a branch, HEAD is overwritten and the detached chain has no ref pointing to it — it becomes unreachable from all named references.

Recovery before switching away:

cat .pes/HEAD                            
echo <hash> > .pes/refs/heads/recovered
echo "ref: refs/heads/recovered" > .pes/HEAD
Recovery after switching away: the commit objects still exist in the object store. Walk every file under .pes/objects/, call object_read on each, and look for OBJ_COMMIT objects whose parent chain leads to your lost work. Git calls this git fsck --lost-found. The GC grace period  keeps the objects safe for at least two weeks.

### Garbage Collection and Space Reclamation

**Q6.1:** — Algorithm to find and delete unreachable objects
Mark-and-sweep:

Mark phase — build the reachable set:

Enumerate every file under .pes/refs/ and read HEAD to get the GC roots.
For each root commit hash, call object_read. Add its hash, the tree hash, and every blob/subtree hash found by recursively walking the tree to a reachable set. Follow each commit's parent pointer until has_parent == 0.
Data structure: a sorted array of ObjectID (32 bytes each) with binary search — O(n log n) to build, O(log n) per lookup. A hash table gives O(1) average lookup for larger repos.

Sweep phase — delete unreachable objects:

Walk every file under .pes/objects/XX/. Reconstruct the full hash from the directory name and filename, convert with hex_to_hash. If not in reachable, call unlink().

Estimate for 100,000 commits, 50 branches:

Assuming roughly four unique objects per commit on average (one commit, one to two trees, one to two new blobs, the rest shared):

Reachable set ≈ 400,000 objects → 400,000 object_read calls in the mark phase.
Sweep: walk all ~400,000 files in the object store.
Total: roughly 800,000 file accesses.




