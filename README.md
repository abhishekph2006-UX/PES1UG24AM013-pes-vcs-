PES-VCS Lab Report

Name:Abhishek P H

SRN:PES1UG24AM013


Phase 1 — Object Storage Foundation
Files modified: object.c

object_write prepends a "<type> <size>\0" header to the data, hashes the whole thing with SHA-256, skips writing if the object already exists (deduplication), creates the shard directory, writes to a temp file, fsyncs, and renames atomically. This guarantees the object store is never left with a partial file even on a crash.

object_read reverifies the SHA-256 after reading (integrity check), parses the type and declared size from the header, validates the declared size against the actual byte count, then returns the data portion in a caller-owned buffer.

Screenshot 1A — ./test_objects


<img width="1470" height="230" alt="Screenshot 2026-04-21 103406" src="https://github.com/user-attachments/assets/1752bb9a-08d6-4bff-8a1e-2321918adab4" />

Screenshot 1B — find .pes/objects -type f


<img width="1202" height="131" alt="Screenshot 2026-04-21 103450" src="https://github.com/user-attachments/assets/65f97194-b759-4435-84da-7aed36ea4dd6" />

Phase 2 — Tree Objects
Files modified: tree.c, Makefile

tree_from_index heap-allocates the Index struct (it is ~5.6 MB, which exceeds the safe stack budget), loads the index, sorts entries by path, then calls the recursive write_tree_level helper.

write_tree_level walks the sorted entries at one directory level. For each plain file entry (no / in the remaining path) it adds a blob TreeEntry. For each directory it groups all entries that share the top-level component, recurses to get the subtree's hash, and adds a 040000 tree entry. After processing all entries it serialises the Tree struct and writes it to the object store.

Screenshot 2A — ./test_tree


<img width="1223" height="200" alt="Screenshot 2026-04-21 112800" src="https://github.com/user-attachments/assets/eea09a98-7175-4a5b-b19b-b992748f686d" />

Screenshot 2B — xxd of a raw tree object


<img width="939" height="235" alt="image" src="https://github.com/user-attachments/assets/6ff69a29-c57c-4d3b-a89c-4a85b3aef10b" />

Phase 3 — The Index (Staging Area)
Files modified: index.c

index_load opens .pes/index with fopen("r") and parses each line with fscanf using the format %o %64s %llu %u %511s. A missing index file is treated as an empty staging area, not an error.

index_save makes a heap-allocated sorted copy (again to avoid a large stack frame), writes to a .tmp file, calls fflush + fsync for durability, then renames atomically.

index_add reads the file, stores it as OBJ_BLOB, stats the file for mtime/size metadata used for fast change detection, upserts the entry, and calls index_save.

Screenshot 3A — pes init → pes add → pes status


<img width="668" height="374" alt="3A" src="https://github.com/user-attachments/assets/8d7a9e08-71ad-4436-8b07-a23d923c1758" />


Screenshot 3B — cat .pes/index


<img width="737" height="64" alt="3B" src="https://github.com/user-attachments/assets/b17e5f7c-9265-4718-b110-6a9520676faf" />

Phase 4 — Commits and History
Files modified: commit.c

commit_create calls tree_from_index to snapshot the staged state, reads the current HEAD to find the parent commit (absent for the first commit), fills a Commit struct with author (PES_AUTHOR env var), Unix timestamp, and message, serialises it with commit_serialize, stores it via object_write(OBJ_COMMIT), then calls head_update to advance the branch pointer atomically.

Screenshot 4A — pes log with three commits


<img width="605" height="124" alt="4A" src="https://github.com/user-attachments/assets/783d74c8-a555-4173-a3f7-d382d8a6026e" />

Screenshot 4B — find .pes -type f | sort


<img width="619" height="119" alt="4B" src="https://github.com/user-attachments/assets/edc6a6eb-032f-4be8-9618-db507a925a6d" />

Screenshot 4C — Reference chain


<img width="578" height="76" alt="4C" src="https://github.com/user-attachments/assets/2268f109-ede8-4fac-bb1f-5920d85c8bde" />

Final — make test-integration


<img width="679" height="452" alt="test 1" src="https://github.com/user-attachments/assets/0e210c8d-9af3-4de8-a9d0-181cc7540a0f" />
<img width="610" height="460" alt="test 2" src="https://github.com/user-attachments/assets/6d92e76a-59a5-4b03-9f6e-dfe05915e3ec" />
<img width="703" height="295" alt="test 3" src="https://github.com/user-attachments/assets/012bf791-6dba-437d-9e14-10a9561c0815" />

Phase 5 — Branching and Checkout

**Q5.1: How would `pes checkout <branch>` work?**

A branch is only a file inside `.pes/refs/heads/branchname` containing the latest commit hash.

When `pes checkout <branch>` is run:

1. Read `.pes/refs/heads/<branch>` to get the commit hash.
2. Change `.pes/HEAD` from:

```text id="m8q2xp"
ref: refs/heads/main
```

to:

```text id="u5n7rv"
ref: refs/heads/<branch>
```

3. Read the commit object pointed to by that branch.
4. Read the tree object stored in that commit.
5. Compare the tree with the current working directory.
6. Update the working directory:

   * Create files that exist in the new branch but not in the current one.
   * Delete files that are not present in the new branch.
   * Replace changed files with the blob contents from the new branch.
7. Update `.pes/index` so that it matches the checked-out branch.

The operation is complex because the working directory may already contain modified files. If checkout blindly overwrites them, the user's work would be lost. Nested directories, deleted files, permissions, and untracked files also make checkout difficult.

---

**Q5.2: How would you detect a dirty working directory conflict?**

Before switching branches:

1. Read the current `.pes/index`.
2. Read the target branch's commit and tree.
3. For every file in the index:

   * Compare the current working-directory file with the staged hash in the index.
   * If the file's contents no longer match the staged blob, then the file has uncommitted changes.
4. Also compare the same file in the target branch's tree:

   * If the target branch contains a different blob hash for that file, checkout would overwrite the user's changes.

In that case, checkout must stop and print an error such as:

```text id="k3m8qp"
error: local changes to 'file.txt' would be overwritten by checkout
```

The algorithm is:

* Current file differs from index = user modified it
* Target branch version differs from current branch version = checkout wants to replace it
* Therefore conflict → refuse checkout

This can be done using only the index and object store hashes, without needing diff algorithms.

---

**Q5.3: What happens in Detached HEAD state?**

Normally `.pes/HEAD` contains a branch reference such as:

```text id="x2m7qp"
ref: refs/heads/main
```

In detached HEAD state, `.pes/HEAD` contains a commit hash directly:

```text id="n2m7rv"
18254094ed54e106ce44799fde551899983179bedf5f259f3044c2906053e4c2
```

Now new commits do not update any branch. Each new commit points to the previous commit, but no branch name points to them.

So the commits still exist, but they are "dangling" and can become unreachable later.

Example:

```text id="p7m2xq"
main -> C3
HEAD detached at C2

new commit C4

HEAD -> C4
main still -> C3
```

If the user later switches back to `main`, commit `C4` is no longer referenced by any branch.

To recover those commits, the user can create a new branch pointing to the detached commit:

```text id="r4n7kv"
pes branch recovered C4
```

or manually create a ref file in `.pes/refs/heads/recovered` containing the commit hash.

Then the commit becomes reachable again and will not be lost.

6 Garbage Collection and Space Reclamation
**Q6.1: Algorithm for Garbage Collection**

To find unreachable objects:

1. Start from every branch in `.pes/refs/heads/`
2. Read the commit hash stored in each branch file.
3. Perform a graph traversal:

   * Mark the commit as reachable
   * Read that commit object
   * Follow its parent commit (if any)
   * Follow its tree object
   * From the tree, follow all blob and subtree hashes recursively
4. Store every visited hash in a hash set.
5. After traversal finishes, scan all files inside `.pes/objects/`.
6. If an object's hash is not present in the reachable set, delete that object file.

Efficient data structure:

* Use a hash table / hash set of object hashes.
* Lookup is O(1), so checking whether an object is already visited is fast.

Pseudo-idea:

```text id="m8q2xp"
reachable = hash_set()

for each branch:
    walk(commit)

walk(commit):
    if commit already in reachable: return
    add commit
    walk(parent)
    walk(tree)

walk(tree):
    add tree
    for each entry:
        if blob: add blob
        if subtree: walk(subtree)
```

For 100,000 commits and 50 branches:

* Many branches share history, so most commits are visited only once.
* Rough estimate:

  * 100,000 commit objects
  * Around 100,000 tree objects
  * Perhaps 200,000–500,000 blob objects depending on project size

So garbage collection may need to visit roughly:

```text id="u5n7rv"
300,000 to 700,000 objects
```

The important point is that shared history prevents revisiting the same commit many times.

---

**Q6.2: Why concurrent GC is dangerous**

Garbage collection can race with a commit operation.

Example race:

1. Commit process creates a new blob object and a new tree object.
2. The commit object has not yet been written.
3. The branch ref still points to the old commit.
4. Garbage collection starts and scans reachable objects from the branch refs.
5. The new blob and tree are not reachable yet, because no commit references them.
6. GC deletes those "unreachable" objects.
7. The commit process then writes a commit that points to the deleted tree/blob.

Now the repository contains a commit whose referenced objects are missing, so history is corrupted.

Example timeline:

```text id="k3m8qp"
Commit process: create blob B
Commit process: create tree T -> B

GC runs:
reachable does not include B or T
GC deletes B and T

Commit process: create commit C -> T
```

Now commit `C` points to a tree that no longer exists.

Git avoids this problem in several ways:

1. New objects are written before refs are updated.
2. GC does not immediately delete new unreachable objects.
3. Git keeps recently created unreachable objects for a grace period (usually 2 weeks).
4. Git uses lock files while updating refs and during GC.
5. Git can stop GC while another process is actively writing objects.

Because of the grace period and locking, objects created by an in-progress commit are not deleted before the commit becomes reachable.





