# Purge Files from Git

This page covers the process involved in purging large files from the Git history of the E3SM Diagnostics project.
These steps can be applied to any repository.

The `e3sm-diags` repo has grown to over 1 GB, making cloning very slow and cumbersome.

Resolves GitHub Issue: <https://github.com/E3SM-Project/e3sm_diags/issues/306>

## Tool of Choice: BFG Repo-Cleaner

> The BFG is a simpler, faster alternative to git-filter-branch for cleansing bad data out of your Git repository history:
>
> - Removing Crazy Big Files
> - Removing Passwords, Credentials & other Private data
>
> The git-filter-branch command is enormously powerful and can do things that the BFG can't - but the BFG is much better for the tasks above, because:
>
> - Faster : 10 - 720x faster
> - Simpler : The BFG isn't particularily clever, but is focused on making the above tasks easy
> - Beautiful : If you need to, you can use the beautiful Scala language to customise the BFG. Which has got to be better than Bash scripting at least some of the time.
>
> &mdash; <https://rtyley.github.io/bfg-repo-cleaner/>

## What to expect

- This is a disruptive action that requires close coordination with collaborators, especially with many forks and clones
- Large unnecessary OUTPUT files (e.g. `.nc`,`.png`, `pdf`, `.tar`) are purged from the repository, effectively reducing the repo's `size-pack`
- BFG rewrites Git commits, and therefore the project's Git history
  - From the first commit you rewrite, every subsequent commit will then differ and have a different hash regardless if it was rewritten or not, since the parent hash will differ from that point forward
- Git will complain about a divergence between upstream and forked repos since Git tracks commits via commit-hashes
  - If any factor in a commit changes, the commit-hash will change when that commit is rewritten during a cleanup
  - Factors: _author, committer, date/time, commit messages, file contents, parent commit_ - Git compares the common/new commits between upstream and fork and sees the history was written, resulting in a divergence
- **The solution to the divergence is found under the "Collaborator Actions" section**

## Step-by-step

1. Clone Repo

   ```bash
   # Using the `mirror` flag clones the bare repo, which means your normal files won't be visible, but it is a full copy of the Git database of your repository
   # It will also update ALL refs on your remote server
   git clone --mirror https://github.com/tomvothecoder/e3sm_diags.git
   ```

2. Backup Repo

   ```bash
   # https://stackoverflow.com/a/54040382
   cd e3sm_diags.git
   git bundle create e3sm_diags.bundle --all
   ```

3. Check Current Repo Size with `git-sizer`

   ```bash
   # https://github.com/github/git-sizer/#getting-started
   $ brew install git-sizer
   $ git-sizer --verbose
   Processing blobs: 15246
   Processing trees: 9572
   Processing commits: 3035
   Matching commits to trees: 3035
   Processing annotated tags: 0
   Processing references: 38
   | Name                         | Value     | Level of concern               |
   | ---------------------------- | --------- | ------------------------------ |
   | Overall repository size      |           |                                |
   | * Commits                    |           |                                |
   |   * Count                    |  3.04 k   |                                |
   |   * Total size               |   843 KiB |                                |
   | * Trees                      |           |                                |
   |   * Count                    |  9.57 k   |                                |
   |   * Total size               |  3.53 MiB |                                |
   |   * Total tree entries       |  85.9 k   |                                |
   | * Blobs                      |           |                                |
   |   * Count                    |  15.2 k   |                                |
   |   * Total size               |  1.62 GiB |                                |
   | * Annotated tags             |           |                                |
   |   * Count                    |     0     |                                |
   | * References                 |           |                                |
   |   * Count                    |    38     |                                |
   |                              |           |                                |
   | Biggest objects              |           |                                |
   | * Commits                    |           |                                |
   |   * Maximum size         [1] |  1.13 KiB |                                |
   |   * Maximum parents      [2] |     2     |                                |
   | * Trees                      |           |                                |
   |   * Maximum entries      [3] |   864     |                                |
   | * Blobs                      |           |                                |
   |   * Maximum size         [4] |  99.4 MiB | **********                     |
   |                              |           |                                |
   | History structure            |           |                                |
   | * Maximum history depth      |   999     |                                |
   | * Maximum tag depth          |     0     |                                |
   |                              |           |                                |
   | Biggest checkouts            |           |                                |
   | * Number of directories  [5] |  2.44 k   | *                              |
   | * Maximum path depth     [6] |     8     |                                |
   | * Maximum path length    [5] |   169 B   | *                              |
   | * Number of files        [5] |  13.6 k   |                                |
   | * Total size of files    [5] |  1.73 GiB | *                              |
   | * Number of symlinks         |     0     |                                |
   | * Number of submodules       |     0     |                                |

   [1]  a30fee3e12792ed745c56dcfec31752764393352
   [2]  6d4601db65443fc547825e496b5edea9636f030b (refs/heads/master)
   [3]  fe6249abbaa449c6592f5dacac887ce49a9fa676 (892dd59884f5fbaba1708c75b6a57545f7750448:acme_diags/plotset5/set5_model_to_model)
   [4]  002235a12193f7231a02a753b66947a496c6ebb7 (refs/remotes/upstream/more_derived_var_HUC2:tests/system/U_201301_201312.nc)
   [5]  d81a217f9a4584678a0062131d696f2f3a571f4a (892dd59884f5fbaba1708c75b6a57545f7750448^{tree})
   [6]  20e4c1535c37c84a69802c44af6a06bc432980c4 (a5d0ee9058cb19e69e061d65f296fe8d12f70bf2^{tree})
   ```

4. Install BFG Repo-Cleaner (Mac)

   ```bash
   # https://docs.github.com/en/github/authenticating-to-github/removing-sensitive-data-from-a-repository
   # https://formulae.brew.sh/formula/bfg
   brew install bfg
   ```

5. Export Git History File Sizes (Blobs)

   ```bash
   # Get current commit hash to name the file
   $ git rev-parse --short HEAD
   6d4601db

   # https://stackoverflow.com/a/42544963
   # This shell script displays all blob objects in the repository, sorted from smallest to largest.
   # Update hash in filename with above output
   $ git rev-list --objects --all |
         git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
         sed -n 's/^blob //p' |
         sort --numeric-sort --key=2 > e3sm_diags_git_files_6d4601db.csv
   ```

6. Validate Which File Extensions to Remove

   - Use the `git_file_validation.ipynb` notebook to confirm which file extensions to remove
   - In our case, we need to remove `.nc`, `.pdf`, `.png`, and `.tar` files

7. Run BFG Repo-Cleaner

   ```bash
   # Delete all files with extension `.nc`, `.pdf`, `.png`, or `.tar`
   # Note, HEAD commit is protected so existing matched files will persist (the ones we need)
   bfg --delete-files "{*.nc,*.pdf,*.png,*.tar}" ../e3sm_diags.git
   ```

8. Examine the Repo and Strip Out Unwanted Data

   ```bash
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   ```

9. Push Rewritten Commits

   ```bash
   git push --force
   ```

10. Check Updated Repo Size with `git-sizer`

    ```bash
    git-sizer --verbose
    ```

### Collaborator Actions

#### Sync Git history with upstream

Now that the large files have been purged, you need to sync the Git history between your fork and upstream.

**If you don't, you will see a divergence. Your fork will be "ahead" and "behind" _n_ # of commits based on the first dirty commit.**

To fix this, you have two options:

1. Replace your fork's master with upstream master

   - **Recommended** approach since we follow the squash and rebase workflow
   - **Must use** this option if you have active branches, or want to avoid having to re-clone repositories

   ```bash
   # https://stackoverflow.com/questions/35214238/replaced-text-with-bfg-and-now-it-shows-my-fork-is-several-hundred-commits-ahea
   # https://e3sm-project.github.io/e3sm_diags/_build/html/master/install.html#b-development-environment
   git remote add upstream https://github.com/E3SM-Project/e3sm_diags.git
   git fetch upstream

   # Must not be on origin master in order to delke
   git checkout <non-master branch>
   git branch -D master

   # Replace fork master with upstream master
   git checkout upstream/master
   git checkout -b master

   # Push new fork master
   git push --set-upstream origin master --force

   # Now rebase any and all existing branches on your fork to the new fork master
   git checkout <non-master-branch>
   git rebase master
   ```

2. Delete existing fork on GitHub and perform fresh fork
   - You'll need to clone the fork and configure it with upstream again
   - **Not recommended** if you have active work on your branches

### Future Actions

- Most of the repo's remaining size is from the `docs/_build_old` folder
- Action: Keep `_build_old` exclusive on `gh-pages` branch and update workflow to **not delete** that directory

### Additional Commands

Restore repo backup

```bash
# https://stackoverflow.com/a/17030169
$ git bundle verify $somewhere/foo.bundle
$ git clone $somewhere/foo.bundle
Cloning into 'foo'...
Receiving objects: 100% (10133/10133), 82.03 MiB | 74.25 MiB/s, done.
Resolving deltas: 100% (5436/5436), done.
$ cd foo
$ git status
```
