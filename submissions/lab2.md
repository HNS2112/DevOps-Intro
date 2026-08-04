# Lab 2 — Version Control Deep Dive: Internals, Recovery, Rebase

**Author:** HNS ([@HNS2112](https://github.com/HNS2112))
**Date:** 4 August 2026
**Environment:** Ubuntu 24.04 (noble), Git 2.54.0, Go 1.26.5

> Note on author names: commits made before this lab carry `user.name = Elvira`;
> it was later changed to `HNS` to match the GitHub handle. The name is baked into
> the commit object and covered by its signature, so older commits cannot show the
> new name without being rewritten. The signing key and committer email are
> unchanged.

---

## Task 1 — Git Object Model + Reflog Recovery

### 1.1 Plumbing chain: HEAD → tree → blob → file

```console
$ git rev-parse HEAD
0ecd6ba6ff0e80f8f23f509c122d0247da0f5584

$ git cat-file -t HEAD
commit

$ git cat-file -p HEAD
tree db88ba097f711cb5246801d5d4aac3b4792e4e7e
parent 9f41b7deb32343a831b5e47c61533fbc7c0ce67d
author Elvira <239804565+HNS2112@users.noreply.github.com> 1785785467 +0300
committer Elvira <239804565+HNS2112@users.noreply.github.com> 1785785467 +0300
gpgsig -----BEGIN SSH SIGNATURE-----
 U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAgmWbLJSeWlD1Vuj7EC1J04q9jqj
 ZyCXUjnh3hTPKxQSYAAAADZ2l0AAAAAAAAAAZzaGE1MTIAAABTAAAAC3NzaC1lZDI1NTE5
 AAAAQIrYi8c4UDJc+sld4lU7nqv/4/mC/213oU4fb4EMK+lFtN5a17TM5FwYfRXgVi5NLq
 5pwGh37MiPy6pqg9mCMwM=
 -----END SSH SIGNATURE-----

docs: add PR template; ignore runtime data dir

Signed-off-by: Elvira <239804565+HNS2112@users.noreply.github.com>

$ git cat-file -p db88ba097f711cb5246801d5d4aac3b4792e4e7e
040000 tree d0f15a494317a8a43f617b9d4784429b9c5167ab	.github
100644 blob 966b15200a257d9978e6fde44ede0423c0224c7b	.gitignore
100644 blob d10c04c6e7e0014f4fe883599c11747c15012d4e	README.md
040000 tree 7d0898a908e274ea809722844cdbd836f3b1c05a	app
040000 tree f4f047dd07b128eda5f899dfdaaf193f0291eaa2	labs
040000 tree c0ac2d55cf4335df659b347df3d19d0594a06b6c	lectures

$ git cat-file -p 966b15200a257d9978e6fde44ede0423c0224c7b
# (.gitignore contents — first and last lines shown)
# ⚠️  KEEP THIS FILE MINIMAL.
...
refs/
app/quicknotes
app/data/
...
data/
app/data/
```

Four observations from this chain:

1. **The commit object holds metadata, not content.** It points at exactly one
   tree, names its parent, and carries author, committer, and the SSH signature
   inline. The `gpgsig` header covers the author lines too, which is why an author
   name cannot be edited after the fact without invalidating the signature and
   producing a different SHA.
2. **The mode column is not a POSIX permission set.** `100644` is a regular file,
   `040000` a tree; Git only distinguishes executable (`100755`) from
   non-executable. Everything else is decided by umask at checkout time.
3. **`submissions/` does not appear in this tree.** The commit is on `main`, and
   that directory exists only on `feature/lab1`. A tree is a snapshot of one
   commit's state, not a listing of the working directory.
4. **The last two lines of the blob are my own change** from Lab 1 — `data/` and
   `app/data/` were added after QuickNotes wrote runtime state into the repo. The
   blob is content-addressed, so those two lines are part of what makes this SHA
   what it is.

### 1.2 Inside `.git/`

```console
$ ls -la .git/
total 52
-rw-rw-r-- 1 hns hns  403 config
-rw-rw-r-- 1 hns hns   73 description
-rw-rw-r-- 1 hns hns  588 FETCH_HEAD
-rw-rw-r-- 1 hns hns   21 HEAD
drwxrwxr-x 2 hns hns 4096 hooks
-rw-rw-r-- 1 hns hns 3183 index
drwxrwxr-x 2 hns hns 4096 info
drwxrwxr-x 3 hns hns 4096 logs
drwxrwxr-x 4 hns hns 4096 objects
-rw-rw-r-- 1 hns hns  573 packed-refs
drwxrwxr-x 5 hns hns 4096 refs

$ cat .git/HEAD
ref: refs/heads/main

$ ls .git/refs/heads/
main

$ ls .git/objects/ | head
info
pack

$ find .git/objects -type f | wc -l
3
```

Interpretation:

- **`HEAD` is a one-line symbolic reference**, not a commit ID. Detaching HEAD
  (which `git bisect` does on every step) replaces this line with a raw SHA.
- **`objects/` contains no two-character directories at all.** All 292 objects
  from the clone are packed; the three files counted are the `.pack`, its `.idx`,
  and `.rev`. Loose objects only appear once I start committing locally — which
  is exactly what happened in the next step.
- **`refs/heads/` lists one branch, though the fork has more.** The rest live in
  `packed-refs`, which Git uses to avoid one tiny file per ref:

```console
$ cat .git/packed-refs
f0c9243b7c80ebb930a1ce7048a1d65b4c2ac493 refs/remotes/origin/bug/bisect-me
42a86fa5180ef5f1a65b5e4096563a37a2558134 refs/remotes/origin/feature/lab1
0ecd6ba6ff0e80f8f23f509c122d0247da0f5584 refs/remotes/origin/main
...
eed564b60e4c7cebccfb586a451a736c54d4f6f9 refs/tags/v0.0.1
^0ec87b808ae6a257a98ecea4a3c8d38a7f2c5ac7
```

The `^` line under `v0.0.1` is the peeled target: `v0.0.1` is an annotated tag —
an object in its own right — and the caret line records the commit it points to.

### 1.3 Simulated disaster and reflog recovery

Two commits, then a destructive reset:

```console
$ git log --oneline -3
1fa1ee0 wip(lab2): more progress
544a3cd wip(lab2): start
0ecd6ba docs: add PR template; ignore runtime data dir

$ git reset --hard HEAD~2
HEAD is now at 0ecd6ba docs: add PR template; ignore runtime data dir

$ git log --oneline -3
0ecd6ba docs: add PR template; ignore runtime data dir
9f41b7d docs(lab7): make seed.json shipping explicit; require bonus artifacts, not logs
8de962e docs(lab11): fix nixpkgs pin vs go.mod collision; add network fallback pitfalls

$ git reflog
0ecd6ba HEAD@{0}: reset: moving to HEAD~2
1fa1ee0 HEAD@{1}: commit: wip(lab2): more progress
544a3cd HEAD@{2}: commit: wip(lab2): start
0ecd6ba HEAD@{3}: checkout: moving from main to main
0ecd6ba HEAD@{4}: clone: from github.com:HNS2112/DevOps-Intro.git
```

Recovery:

```console
$ git reset --hard 1fa1ee0
HEAD is now at 1fa1ee0 wip(lab2): more progress

$ git log --oneline -3
1fa1ee0 wip(lab2): more progress
544a3cd wip(lab2): start
0ecd6ba docs: add PR template; ignore runtime data dir

$ cat submissions/lab2.md
important work
more important work
```

`git log` showed nothing because no branch pointed at those commits any more —
but the objects were never deleted. The reflog is a per-repository log of where
`HEAD` has been, so `HEAD@{1}` still named `1fa1ee0` and the whole chain was
reachable through it.

The reflog also caught a mistake: line `HEAD@{3}` reads `checkout: moving from
main to main`, which shows that my `git switch -c feature/lab2` never ran and
both commits landed on `main`. The branch was created at recovery time instead,
which left `main` clean.

**What if `git gc` had run in between?** In the default configuration, nothing:
`gc` treats reflog entries as roots, and `gc.reflogExpire` keeps reachable
entries for 90 days and unreachable ones for 30, so the "lost" commits stay
protected. The danger appears when that safety net is absent — a fresh CI clone
has an effectively empty reflog, and an explicit
`git reflog expire --expire=now --all` followed by `git gc --prune=now` deletes
the objects for real. That is why the lab's advice is to capture the SHA before
experimenting rather than to trust the window.

---

## Task 2 — Signed Tag and Rebase

### 2.1 Annotated, signed release tag

```console
$ git tag -l --format='%(refname:short) %(objecttype) %(*objecttype)'
v0.0.1 tag commit
v0.1.0-lab2-HNS2112 tag commit

$ git cat-file -t v0.1.0-lab2-HNS2112
tag

$ git tag -v v0.1.0-lab2-HNS2112
Good "git" signature for 239804565+HNS2112@users.noreply.github.com with ED25519 key SHA256:10QjC3lakxoG0ObZMeXhw0fBga6g+mk9UyhPWJNGmC0
object 0ecd6ba6ff0e80f8f23f509c122d0247da0f5584
type commit
tag v0.1.0-lab2-HNS2112
tagger HNS <239804565+HNS2112@users.noreply.github.com> 1785861815 +0300

Lab 2 milestone — version control deep dive

$ git push origin v0.1.0-lab2-HNS2112
 * [new tag]         v0.1.0-lab2-HNS2112 -> v0.1.0-lab2-HNS2112
```

`git cat-file -t` returning `tag` rather than `commit` is what separates an
annotated tag from a lightweight one: it is a real object with its own author,
message, and signature, and only annotated tags can be signed at all.

The lab handout names the tag `v0.1.0-lab2-${USER}`. `$USER` is the local login
name (`hns` here), which would not identify me in the course repo, so the tag was
named after the GitHub handle explicitly.

### 2.2 Branch protection blocked the rebase setup

Step 2.2 asks for a commit pushed straight to `main`. My fork has the branch
protection from Lab 1's bonus still enabled, so the push was rejected:

```console
$ git commit -S -s --allow-empty -m "docs: upstream moved while you worked"
[main 8768460] docs: upstream moved while you worked

$ git push origin main
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote:
remote: - Changes must be made through a pull request.
To github.com:HNS2112/DevOps-Intro.git
 ! [remote rejected] main -> main (protected branch hook declined)
```

Worth noting which rule fired: only the pull-request requirement. The signature
rule stayed silent because the commit *was* signed. In Lab 1 the same push
tripped both rules at once, so this is a clean demonstration that they are
evaluated independently rather than as one gate.

Two lab requirements genuinely conflict here — Lab 1 asks for a branch that
cannot be pushed to directly, Lab 2 asks to push to it. I relaxed the rule
narrowly: `required_pull_request_reviews` and `enforce_admins` off, signature and
linear-history requirements left in place. The push then succeeded, which
confirms the signing rule was still active and still satisfied. Protection was
restored immediately afterwards:

```console
$ gh api "/repos/HNS2112/DevOps-Intro/branches/main/protection" -q '...'
{"admins":true,"linear":true,"pr":1,"signatures":true}
```

### 2.3 Rebase, before and after

```console
$ git log --oneline --graph -5      # before
* 1fa1ee0 wip(lab2): more progress
* 544a3cd wip(lab2): start
* 0ecd6ba docs: add PR template; ignore runtime data dir
* 9f41b7d docs(lab7): make seed.json shipping explicit; require bonus artifacts, not logs
* 8de962e docs(lab11): fix nixpkgs pin vs go.mod collision; add network fallback pitfalls

$ git rebase origin/main
Successfully rebased and updated refs/heads/feature/lab2.

$ git log --oneline --graph -5      # after
* 1d91a02 wip(lab2): more progress
* 7607087 wip(lab2): start
* 8768460 docs: upstream moved while you worked
* 0ecd6ba docs: add PR template; ignore runtime data dir
* 9f41b7d docs(lab7): make seed.json shipping explicit; require bonus artifacts, not logs

$ git log --format='%h %G? %s' -4
1d91a02 G wip(lab2): more progress
7607087 G wip(lab2): start
8768460 G docs: upstream moved while you worked
0ecd6ba U docs: add PR template; ignore runtime data dir
```

Both of my commits changed SHA — `1fa1ee0 → 1d91a02` and `544a3cd → 7607087` —
while their content stayed identical. A commit's hash covers its parent, so
replaying it onto a new base necessarily produces a new object. Rebase creates
commits, it does not move them.

Both replayed commits still verify (`G`): Git re-signed them during the replay
with the key configured on this machine. `0ecd6ba` shows `U` only because it was
signed on a different machine whose public key is not in this host's
`allowed_signers`; GitHub shows it as Verified, since all three of my keys are
registered there.

The push used `--force-with-lease`, which refuses to overwrite the remote if it
has moved since my last fetch. Plain `--force` would have overwritten it blindly.

### Merge vs rebase

The distinction I apply is whether the history is shared yet.

**Rebase** while the work is still mine — a feature branch nobody else has pulled.
It keeps history linear, makes `git bisect` meaningful (as in this lab's bonus:
every commit is a real state of the code, not a merge of two states), and avoids
merge commits that carry no information. The cost is that every replayed commit
is a new object, so anyone who already pulled the branch is now out of sync.

**Merge** once the history is shared, and for integrating a completed branch into
`main`. The merge commit is a fact worth recording: it says two lines of work
converged at this point, and it never rewrites what someone else may have based
work on.

Concretely for this course: rebase `feature/labN` onto `main` before opening the
PR, then let the PR merge — and never rebase anything already pushed to a branch
someone else might be building on.

---

## Bonus Task — Bisecting a Real Bug

### B.1–B.2 Automated bisect

```console
$ git switch -c bisect-quickn upstream/bug/bisect-me
$ git bisect start
$ git bisect bad HEAD
$ git bisect good v0.0.1
Bisecting: 1 revision left to test after this (roughly 1 step)

$ git bisect run sh -c 'cd app && go test ./... && go build ./...'
running 'sh' '-c' 'cd app && go test ./... && go build ./...'
--- FAIL: TestStore_PersistsAcrossReload (0.00s)
    store_test.go:78: nextID not restored: got 1, want 2
FAIL
FAIL	quicknotes	0.003s
Bisecting: 0 revisions left to test after this (roughly 0 steps)
running 'sh' '-c' 'cd app && go test ./... && go build ./...'
ok  	quicknotes	0.004s
f285ede8611e55ac0a7d01100891c0cc775e0709 is the first bad commit
```

```console
$ git bisect log
git bisect start
# bad: [f0c9243b7c80ebb930a1ce7048a1d65b4c2ac493] docs(app): mention go test invocation
git bisect bad f0c9243b7c80ebb930a1ce7048a1d65b4c2ac493
# good: [0ec87b808ae6a257a98ecea4a3c8d38a7f2c5ac7] chore(app): document versioning scheme (bisect fixture baseline)
git bisect good 0ec87b808ae6a257a98ecea4a3c8d38a7f2c5ac7
# bad: [f285ede8611e55ac0a7d01100891c0cc775e0709] refactor(store): simplify nextID restoration in load()
git bisect bad f285ede8611e55ac0a7d01100891c0cc775e0709
# good: [cb89bb9ee2ee5010b166061447eaca3ae0da2378] docs(store): comment the load() decode step
git bisect good cb89bb9ee2ee5010b166061447eaca3ae0da2378
# first bad commit: [f285ede8611e55ac0a7d01100891c0cc775e0709] refactor(store): simplify nextID restoration in load()
```

### B.3 The offending commit

**SHA:** `f285ede8611e55ac0a7d01100891c0cc775e0709`
**Message:** `refactor(store): simplify nextID restoration in load()`
**Author:** Dmitrii Creed, 5 June 2026

```diff
--- a/app/store.go
+++ b/app/store.go
@@ -51,7 +51,7 @@ func (s *Store) load() error {
 	}
 	for _, n := range notes {
 		s.notes[n.ID] = n
-		if n.ID >= s.nextID {
+		if n.ID > s.nextID {
 			s.nextID = n.ID + 1
 		}
 	}
```

One character, and a textbook off-by-one. When the highest stored ID equals the
current `nextID`, the strict comparison is false and the counter is never
advanced, so the next note created after a reload is assigned an ID that is
already taken. Nothing crashes; the store quietly overwrites an existing note.
The failing assertion — `nextID not restored: got 1, want 2` — is exactly this.

The commit message says "simplify", which is what makes this class of change
dangerous: it reads as a no-op cleanup, so it draws little review attention.

**On log₂(N).** The range here was small — `git rev-list --count` reports 4
commits between the good tag and the bad HEAD — and bisect needed 2 steps, which
matches log₂(4). The saving is invisible at this size, but the scaling is the
point: 1,000 commits need 10 builds instead of ~500 for an average linear scan,
and 1,000,000 need 20. Each step halves the candidate range, so cost grows with
the logarithm of history length rather than with history length.

The more practical gain is `bisect run`. Passing a script turns the search into a
non-interactive operation: the definition of "broken" becomes an executable
predicate rather than a human judgement call, which makes the result reproducible
and lets the same command run unattended in CI. It also forces precision — the
test either fails or it does not, so there is no drift in what is being bisected
for.

---

## Summary

| Task | Status |
|------|--------|
| Task 1 — Object model, reflog recovery, gc reflection | Complete |
| Task 2 — Signed annotated tag, rebase, merge-vs-rebase | Complete |
| Bonus — `git bisect` | Complete |

Artifacts: raw command outputs are committed under `evidence/lab2/`.
