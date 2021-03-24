# Purge Files from Git

This page covers the process involved in purging large files from the Git history of the `e3sm-diags` project.

The `size-pack` of the `e3sm-diags` repo has grown to over 1 GB, making cloning very slow and cumbersome.

- <https://git-scm.com/docs/git-count-objects>

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

1. Clone repo

   ```bash
   # Using the `mirror` flag clones the bare repo, which means your normal files won't be visible, but it is a full copy of the Git database of your repository
   # It will also update ALL refs on your remote server
   git clone --mirror https://github.com/tomvothecoder/e3sm_diags.git
   ```

2. Backup repo

   ```bash
   # https://stackoverflow.com/a/54040382
   cd e3sm_diags.git
   git bundle create e3sm_diags.bundle --all
   ```

3. Check current repo size

   ```bash
   # https://stackoverflow.com/a/16163608
   $ git gc
   $ git count-objects -vH
    count: 0
    size: 0 bytes
    in-pack: 28457
    packs: 1
    size-pack: 1.13 GiB
    prune-packable: 0
    garbage: 0
    size-garbage: 0 bytes
   ```

4. Install BFG Repo-Cleaner (Mac)

   ```bash
   # https://docs.github.com/en/github/authenticating-to-github/removing-sensitive-data-from-a-repository
   # https://formulae.brew.sh/formula/bfg
   brew install bfg
   ```

5. List large commits in history

   ```bash
   # https://stackoverflow.com/a/42544963
   # Exclude --to=iec-i for bytes
   $ git rev-list --objects --all |
         git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
         sed -n 's/^blob //p' |
         sort --numeric-sort --key=2 |
         cut -c 1-12,41- |
         $(command -v gnumfmt || echo numfmt) --field=2 --to=iec-i --suffix=B --padding=7 --round=nearest > e3sm_diags_git_files.csv

    # Subset of output
    7e3db185f9b7  1.0MiB acme_diags/model_to_obs/lat_lon_vector/ERA-Interim/ERA-Interim-TAUY-ANN-global.png
    467088e7260d  1.2MiB tests/system/TREFHT_185001_185012.nc
    302351b31870  1.4MiB acme_diags/test_vector1.png
    5e4138957806  1.7MiB acme_diags/plot/cartopy/test_vector1.png
    7fe0dc7e01e4  1.8MiB acme_diags/oldfiles/newobs_rlut.tar
    945f75dd774c  2.3MiB tests/system/TREFHT_201201_201312.nc
    59894fe5f3fe  6.7MiB tests/T_20161118.beta0.FC5COSP.ne30_ne30.edison_ANN_climo.nc
    6e12ca84343b  6.8MiB tests/system/T_20161118.beta0.FC5COSP.ne30_ne30.edison_ANN_climo.nc
    0de240a9c697  7.6MiB acme_diags/tier1b_cloud.tar
    643365ac8e59  8.3MiB tests/CLD_MISR_20161118.beta0.FC5COSP.ne30_ne30.edison_ANN_climo.nc
    d1fa2696a553  8.4MiB tests/system/CLD_MISR_20161118.beta0.FC5COSP.ne30_ne30.edison_ANN_climo.nc
    fe0c75cc7eab   11MiB acme_diags/plotset5/set5_PRECT_GPCP/GPCP_PRECT_ANN_global.pdf
    7ddd2051268c   11MiB tests/system/ta_ERA-Interim_ANN_198001_201401_climo.nc
    b01c38406fcd   18MiB tests/system/CLDMISR_ERA-Interim_ANN_198001_201401_climo.nc
    10d218439b25   23MiB tests/climo/PRECC_200001_201412.nc
    249a1bc713e4   23MiB tests/climo/PRECL_200001_201412.nc
    7da9c43ef279   40MiB acme_diags/plotset5/set5_ANN_T200_ECMWF/test_lev.png.pdf
    002235a12193   99MiB tests/system/U_201301_201312.nc
   ```

6. Run BFG Repo-Cleaner

   ```bash
   # Delete all files with extension `.nc`, `.pdf`, `.png`, or `.tar`
   # Note, HEAD commit is protected so existing matched files will persist (the ones we need)
   bfg --delete-files "{*.nc,*.pdf,*.png,*.tar}" ../e3sm_diags.git
   ```

7. Examine the repo and strip out unwanted dirty data

   ```bash
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   ```

8. Push rewritten commits

   ```bash
   git push --force
   ```

9. Confirm files removed by comparing hashes

   ```Bash
   # Subset of output from step 6
                            Before     After
    -------------------------------------------
    First modified commit | eec48df7 | 60318b09
    Last dirty commit     | d6072730 | f41d033c
   ```

   First modified commit

   - Upstream: <https://github.com/E3SM-Project/e3sm_diags/tree/eec48df7/>
   - Fork: <https://github.com/tomvothecoder/e3sm_diags/tree/60318b09/>

   Last dirty commit

   - Upstream: <https://github.com/E3SM-Project/e3sm_diags/tree/d6072730/>
   - Fork: <https://github.com/tomvothecoder/e3sm_diags/tree/f41d033c/>

10. Check updated repo size

    ```bash
    $ git gc
    $ git count-objects -vH
    count: 0
    size: 0 bytes
    in-pack: 16778
    packs: 1
    size-pack: 22.00 MiB
    prune-packable: 0
    garbage: 0
    size-garbage: 0 bytes
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
