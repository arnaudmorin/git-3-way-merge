# 3-way merging is confusing for humans

When using the `cherry-pick` workflow (e.g. cherry-picking commits from a branch
to another to push new changes to production), a human is expecting to see the
exact same diff applied from source branch to target branch, and if the patch
cannot be applied, the human expect that it will fail (conflict).

**But it's not failing!**

Here is why.

The problem is that, by default, `git cherry-pick` is using the `3-way` merge
strategy to apply the diff on the target branch.

Here is how to reproduce it.

First, clone this repo on `source` branch:

```bash
git clone -b source https://github.com/arnaudmorin/git-3-way-merge.git
cd git-3-way-merge
```

You will find a `myfile` file in the repo, take a look at it:
```bash
line1

line3
```

Notice that `line2` is missing

Now, take a look at the `patch` from last `commit` on this branch:
```bash
git show HEAD
```

Notice that we removed `line2`, but also `line4`.

Actually, `line4` was introduced in previous commit:
```bash
git show HEAD^
```

So, the last commit is removing 2 lines.

Now, let's go on `target` branch
```bash
git checkout target
```

Take a look at `myfile`
```bash
line1
line2
line3
```

All lines are there, but not `line4`.

Try to `cherry-pick` the `HEAD` commit from `source` branch:
```bash
git cherry-pick source
```

This will succeed, even if `line4` is absent.

Actually, git does not care about removal of `line4`, because it's basically doing a `3-way`
merge between common ancestor of both branches (C), current state of `target`
(A), and the commit comming from `source` (B) to produce the new commit (D):
```
    A -- D
  /
C
  \ 
    B

```

D is supposed to be a cherry-pick of B, so applying the same `patch` as B, over
A. **This is not always true** (it's not here).

(see also https://en.wikipedia.org/wiki/Merge_(version_control) )

In that `3-way` merge, line4 never existed (it was added, then removed in
`source` branch).

So, the actual patch applied on `target` branch is not the same diff as the one
applied on `source`. This can be checked by comparing the `patchid`:

```bash
git show | git patch-id | awk '{print $1}'
061392d34243b2052e3f54e29c86788a8220ad20

git show source | git patch-id | awk '{print $1}'
8930574d635990dd8443ec83af3eefa44d3a3336
```

This can also be done manually without git with `diff3` tool.

We first need to extract `myfile` from differents commits to work on:
```bash
# Extract common ancestor myfile (C)
git checkout $(git merge-base origin/source origin/target) -- myfile && mv myfile C

# Extract target myfile (A)
git checkout origin/target -- myfile && mv myfile A

# Extract source myfile (B)
git checkout origin/source -- myfile && mv myfile B

# Now use diff3 to simulate the 3-way merge
diff3 -m A C B
```

This will produce:
```bash
line1

line3
```

which is exactly what `git` did when cherry-picking the `source` commit on
`target` branch...


If you want to apply exactly the patch, you cannot use the `cherry-pick`
command, but rather the `apply` one:
```bash
# Go back to origin/target branch first
git reset --hard origin/target

# Now apply the change from source, without 3-way merge, but more like `patch` way
git show source | git apply
```
