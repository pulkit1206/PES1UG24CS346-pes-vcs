# PES-VCS Lab Report

## Phase 5: Branching and Checkout

### Q5.1: How would you implement `pes checkout <branch>`?
To implement `pes checkout <branch>`, the following steps are required:
1. **Verify Branch Existence:** Check if `.pes/refs/heads/<branch>` exists.
2. **Read Target Tree:** Read the commit hash from the branch file, then read the commit object to find its root tree hash.
3. **Update Working Directory:**
   - This is the most complex part. The working directory must be synchronized with the target tree.
   - Files present in the target tree but not in the current working directory must be created.
   - Files present in the current working directory but not in the target tree must be deleted (if they are tracked).
   - Files that differ must be overwritten.
4. **Update the Index:** The index must be updated to match the target tree exactly, so that `pes status` shows no changes.
5. **Update HEAD:** Change `.pes/HEAD` to point to the new branch (e.g., `ref: refs/heads/<branch>`).

**Complexity:** The operation is complex because it must handle **uncommitted changes**. If a user has modified a file that also differs between the current and target branches, a naive checkout would overwrite the user's work. It must also handle directory structures (creating/removing subdirectories).

### Q5.2: Detecting a "Dirty Working Directory"
A "dirty" conflict can be detected by comparing three states:
1. **HEAD Tree:** The state of the last commit.
2. **Index:** The state of the staging area.
3. **Working Directory:** The actual files on disk.

**Detection Algorithm:**
- If `Working Directory File != Index Entry` (based on mtime/size), the file is **modified**.
- If this modified file also differs between the **Current HEAD** and the **Target Branch HEAD**, the checkout must be refused to prevent data loss.
- If the file is the same in both branches, Git can sometimes "carry forward" the local change, but for simplicity, any difference between Index and Working Directory for a file that changes in the target branch should trigger a conflict.

### Q5.3: Detached HEAD
A **Detached HEAD** occurs when `.pes/HEAD` contains a raw commit hash instead of a `ref: refs/heads/...` string.
- **Committing in this state:** Commits will succeed, and the new commit's parent will be the current HEAD hash. HEAD will be updated to the new commit's hash.
- **The Risk:** Since no branch pointer (like `main`) is being updated, if the user checkouts another branch, there will be no reference left pointing to these new commits. They become "orphaned".
- **Recovery:** A user can recover these commits by checking the command output for the last commit hash or using a tool like `git reflog` (if implemented) to find the hash, and then creating a new branch at that hash: `pes branch <new-name> <hash>`.

---

## Phase 6: Garbage Collection and Space Reclamation

### Q6.1: Finding and Deleting Unreachable Objects
**Algorithm (Mark and Sweep):**
1. **Initialization:** Create a set of "reachable" hashes, initially empty.
2. **Mark Phase:**
   - Start from all references in `.pes/refs/` and the hash in `.pes/HEAD`.
   - For each commit hash found:
     - Add to "reachable" set.
     - Recursively visit the commit's `tree` and `parent` hashes.
     - For each tree object, recursively visit all child `tree` and `blob` hashes.
3. **Sweep Phase:**
   - Iterate through all objects in `.pes/objects/`.
   - If an object's hash is NOT in the "reachable" set, delete it.

**Data Structure:** A **Hash Set** (Bloom filter for optimization) is ideal for tracking reachable hashes efficiently.
**Estimate:** For 100,000 commits and 50 branches, you would need to visit at least 100,000 commit objects. However, since trees and blobs are shared, the number of unique objects visited depends on the project's churn. In a large repo, this could be millions of objects.

### Q6.2: GC Race Conditions
**Danger:** Running GC concurrently with a commit is dangerous because a commit operation follows a specific sequence: it writes blobs, then trees, then finally the commit object and updates the ref.
**Race Condition:**
1. Commit process writes a new blob `B1`. `B1` is not yet reachable from any ref.
2. GC starts, scans refs, and determines `B1` is unreachable.
3. GC deletes `B1`.
4. Commit process tries to write a tree object that references `B1`. The repository is now corrupted because it points to a missing blob.

**Git's Solution:** Git uses a **grace period** (typically 2 weeks). GC will only delete unreachable objects that are older than a certain timestamp (using the file's `mtime`). Since a concurrent commit would have just created the object, its `mtime` would be very recent, and GC would skip it.
