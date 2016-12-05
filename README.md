# Subgit Move Handler

This shell script will help you in restoring subgit to a usable state after an `svn move` operation.

It works by applying empty commits to the git repo, and creates subgit mapping references for the svn operation. When subgit is then reinstalled to the git repo, it will think it is synchronized up until the move in SVN.

This way you will get to maintain all of your `git log` data, there will just be an extra empty commit in the logs for the move. Hurray!

# Usage

From inside a local clone of the subgit-managed remote `repo.git`:

```
subgit-move-handler [--pick|--complex]

Options:

    --pick       Select which branches should be synced
    --complex    Specify path and ref details for every branch
```

If you're using the [SVN best practices](http://svn.apache.org/repos/asf/subversion/trunk/doc/user/svn-best-practices.html) layout of `/path/to/project/[trunk,tags,branches]/`, and all branches on the remote should be synced, you can simply run the script inside a locally-cloned copy of the repo:

```
$ git clone https://github.com/squarebracket/subgit-move-handler.git
$ git clone https://git.example.com/myrepo.git
$ cd myrepo && ../subgit-move-handler/subgit-move-handler
```

If you want to pick which branches are synced (for example, if you have an `excludeBranches` in your subgit config), you can provide the `--pick` flag. It will prompt you before every branch.

If you're not following SVN best practices, you can provide the `--complex` flag, which will prompt you for the SVN path location details for every branch. This flag implies `--pick`.

# Step-by-step Guide

Before you use this script on your production repos, I **strongly** recommend testing it using a copy of both the svn repo and the git repo. While I am pretty confident that this script will work without issue, I can't say I tested it in a wide variety of situations.

**Step 1**: Disable subgit on the remote repository by doing `subgit uninstall /path/to/repo.git`. Don't use the `--purge` option, as it will wipe away sync information that we want to keep.

**Step 2**: Clone the remote repo locally.

**Step 3**: Run the script from inside the local repo. It will ask you for the following information:

- The SVN revision number when the move happened, WITHOUT the r (e.g. `4457`)
- The git-formatted commit date-time of the SVN move (e.g. `Fri, 18 Nov 2016 21:53:48 +0000`)
- The SVN commit message used for the move (e.g. `Move a/b to c/b`)
- Pathing information; what it specifically asks for depends on whether you're using `--complex`

If you're _not_ using `--complex`, you will be prompted only once for pathing information -- for which you should provide the path from the SVN repo's root to the location of the new base folder that contains `branches`, `tags`, and `trunk`. Note that this is *not* the path from what you've provided to subgit as the root -- it is the path from the root of the entire SVN repo. For example, if the repo now lives at the URL `svn://example.com/new/path/to/project`, you must type `new/path/to/project`, even though you would pass `svn://example.com/new/path/to/project` to `subgit configure`. Since you're using the standard layout, the script will automatically generate the proper mapping paths for each branch.

If you _are_ using `--complex`, you must manually provide the pathing information for every branch you wish to sync. This requires two inputs per branch:
- The relative path from SVN repo root to the branch in question
  For example, if the branch now lives at `http://example.com/svn/branches/new/path/to/project/branchname`, you would type `branches/new/path/to/project/branchname`
- The relative path from subgit's configured root to the branch in question
  If you are using the `/[branches,tags,trunk]/path/to/project` layout in SVN, this input should be the same as the previous input. If, however, you're doing something truly wacky like `/path/to/division/[branches,tags,trunk]/path/to/project`, then you would provide only the `branches/path/to/project/branchname` part of the SVN path, since the root URL you would pass to `subgit configure` would be `http://example.com/svn/path/to/division/`.

**NOTE**: There is _very_ little validation done on inputs in the script. You should make absolutely sure you have the correct format before running the script.

**Step 4**: Review the log information using `git log --all --decorate --pretty=oneline`. Each git branch that you chose to sync should have a corresponding `refs/svn/root/<svn_path_to_branch>`. Without `--complex`, `svn_path_to_branch` should be `trunk` for master or `branches/<branch>` for every other branch.

**Step 5**: On the remote repo, do `rm -rf /path/to/repo.git/subgit /path/to/repo.git/svn`. This will wipe away any stale subgit configuration or caching that may interfere.

(If you have a very complex subgit configuration that you don't want to wipe away, you should be able to get away with just removing the `svn`, `subgit/.run`, and `subgit/tmp` directories. Modify the SVN URL by hand in `subgit/config` instead of running `subgit configure` in the next step)

**Step 6**: Reinstall subgit using:
```
$ subgit configure http://example.com/svn/new/path/to/project /path/to/repo.git
# Edit configuration as necessary
$ subgit install /path/to/repo.git
```

You should now be properly synchronized.
