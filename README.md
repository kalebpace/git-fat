# Introduction
Checking large binary files into a source repository (Git or otherwise) is a bad idea because repository size quickly becomes unreasonable.
Even if the instantaneous working tree stays manageable, preserving repository integrity requires all binary files in the entire project history, which given the typically poor compression of binary diffs, implies that the repository size will become impractically large.
Some people recommend checking binaries into different repositories or even not versioning them at all, but these are not satisfying solutions for most workflows.

## Features of `git-fat`
* clones of the source repository are small and fast because no binaries are transferred, yet fully functional (unlike `git clone --depth`)
* `git-fat` supports the same workflow for large binaries and traditionally versioned files, but internally manages the "fat" files separately
* `git-bisect` works properly even when versions of the binary files change over time
* selective control of which large files to pull into the local store
* local fat object stores can be shared between multiple clones, even by different users
* can easily support fat object stores distributed across multiple hosts
* depends only on stock Python and rsync

## Related projects
* [git-annex](http://git-annex.branchable.com) is a far more comprehensive solution, but with less transparent workflow and with more dependencies.
* [git-media](https://github.com/schacon/git-media) adopts a similar approach to `git-fat`, but with a different synchronization philosophy and with many Ruby dependencies.

Edit `.gitattributes` to regard any desired extensions as fat files.
    $ cat >> .gitattributes
    ^D

Run `git fat init` to active the extension. Now add and commit as usual.
Matched files will be transparently stored externally, but will appear
complete in the working tree.
Set a remote store for the fat objects by editing `.gitfat`.
Before we start, let's turn on verbose reporting so we can see what's
happening. Without this environment variable, all the output lines
starting with `git-fat` will not be shown.

    $ export GIT_FAT_VERBOSE=1

First, we create a repository and configure it for use with `git-fat`.

    [rsync]

Now we add a binary file whose name matches the pattern we set in `.gitattributes`.


The patch itself is very simple and does not include the binary.

    $ git show --pretty=oneline HEAD
    918063043a6156172c2ad66478c6edd5c7df0217 Add master.tar.gz
    diff --git a/master.tar.gz b/master.tar.gz
    new file mode 100644
    index 0000000..12f7d52
    --- /dev/null
    +++ b/master.tar.gz
    @@ -0,0 +1 @@
    +#$# git-fat 1f218834a137f7b185b498924e7a030008aee2ae

## Pushing fat files
Now let's push our fat files using the rsync configuration that we set up earlier.

    building file list ...

We we might normally set a remote now and push the git repository.

## Cloning and pulling
Now let's look at what happens when we clone.
    $ ls -l                                 # file is just a placeholder
    total 4
    -rw-r--r--  1 jed  users  53 Nov 25 22:42 master.tar.gz
    $ cat master.tar.gz                     # holds the SHA1 of the file
    #$# git-fat 1f218834a137f7b185b498924e7a030008aee2ae

We can always get a summary of what fat objects are missing in our local cache.

    Orphan objects:
    1f218834a137f7b185b498924e7a030008aee2ae

Now get any objects referenced by our current `HEAD`. This command also
accepts the `--all` option to pull full history, or a revision to pull
selected history.

    receiving file list ...
    Restoring 1f218834a137f7b185b498924e7a030008aee2ae -> master.tar.gz

Everything is in place

    $ git status
    git-fat filter-clean: caching to /tmp/repo2/.git/fat/objects/1f218834a137f7b185b498924e7a030008aee2ae
    # On branch master
    nothing to commit, working directory clean
## Summary
* Set the "fat" file types in `.gitattributes`.
* Use normal git commands to interact with the repository without
  thinking about what files are fat and non-fat. The fat files will be
  treated specially.
* Synchronize fat files with `git fat push` and `git fat pull`.

## Implementation notes
The actual binary files are stored in `.git/fat/objects`, leaving `.git/objects` nice and small.

    $ du -bs .git/objects
    2212    .git/objects/
    $ ls -l .git/fat/objects                # This is where the file actually goes, but that's not important
    total 8
    -rw------- 1 jed users 6449 Nov 25 17:01 1f218834a137f7b185b498924e7a030008aee2ae

If you have multiple clones that access the same filesystem, you can make
`.git/fat/objects` a symlink to a common location, in which case all content
will be available in all repositories without extra copies. You still need to
`git fat push` to make it available to others.

# Some refinements
* Allow pulling and pushing only select files
* Relate orphan objects to file system
* More friendly configuration for multiple fat remotes