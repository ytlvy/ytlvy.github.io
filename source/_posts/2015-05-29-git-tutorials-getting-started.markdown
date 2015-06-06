---
layout: post
title: "git tutorials -- Getting Started"
date: 2015-05-29 10:05:34 +0800
comments: true
categories: Git
---
## git tutorials
### setup a repository 
```
// initialize an local repository
git init
git init <directory>


//initialize a Central repositories or server 
git init --bare <directory>.git

ssh <user>@<host>
cd path/above/repo 
git init --bare my-project.git
```

<!--more-->

### clone 
```
git clone <repo>
git clone <repo> <directory>
```

### config
```
//edit config
git config --global --edit
```

#### config options file
1. <repo>/.git/config – Repository-specific settings.
2. ~/.gitconfig – User-specific settings. This is where options set with the --global flag are stored
3. $(prefix)/etc/gitconfig – System-wide settings.

### Saving changes  ---------- edit/stage/commit 
```
git add <file>         // Stage all changes in <file> for the next commit.
git add <directory>    // Stage all changes in <directory> for the next commit
git add p              //  interactive staging
git add .              // stage all changes in current directory
```

### log
```
git log
git log -n <limit>
git log --oneline
git log --stat // include which files were altered and the relative number of lines that modified
git log -p     //  shows the full diff of each commit,
git log --author="<pattern>"
git log --grep="<pattern>"
git log <since>..<until>     //Both arguments can be either a commit ID, a branch name, HEAD
git log <file>
git log --graph --decorate --oneline


git log --author="John Smith" -p hello.py

//The .. syntax is a very useful tool for comparing branches
git log --oneline master..branch
```

commit 3157ee3718e180a9476bf2e5cab8e3f1e78a73b7   ---- SHA-1 checksum of the commit’s contents
The ~ character is useful for making relative references to the parent of a commit. For example, 3157e~1 refers to the commit before 3157e, and HEAD~3 is the great-grandparent of the current commit.


### view old commits -- checkout
checking out files, 
checking out commits
checking out branches. 

Checking out a commit makes the entire working directory match that commit. This can be used to view an old state of your project without altering your current state in any way. Checking out a file lets you see an old version of that particular file, leaving the rest of your working directory untouched.

```
git checkout

git checkout master   // return to master branch

git checkout <commit> <file>

//You can use either a commit hash or a tag as the <commit> argument. 
//This will put you in a detached HEAD state
git checkout <commit>

```

```
git log --oneline
git checkout a1e8fb5
git checkout master


git checkout a1e8fb5 hello.py
git checkout HEAD hello.py

```

### Undo

### git revert 
git revert is a “safe” way to undo changes
Generate a new commit that undoes all of the changes introduced in <commit>, then apply it to the current branch.
Reverting should be used when you want to remove an entire commit from your project history.

```
git revert <commit>
```


### Reverting vs. Resetting
It's important to understand that git revert undoes a single commit—it does not “revert” back to the previous state of a project by removing all subsequent commits. In Git, this is actually called a reset, not a revert.

Reverting has two important advantages over resetting. First, it doesn’t change the project history, which makes it a “safe” operation for commits that have already been published to a shared repository. For details about why altering shared history is dangerous, please see the git reset page.

Second, git revert is able to target an individual commit at an arbitrary point in the history, whereas git reset can only work backwards from the current commit. For example, if you wanted to undo an old commit with git reset, you would have to remove all of the commits that occurred after the target commit, remove it, then re-commit all of the subsequent commits. Needless to say, this is not an elegant undo solution.

```

git commit -m "Make some changes that will be undone
git revert HEAD

```

### git reset
dangerous method

#### use
1. remove committed snapshots, 
2. undo changes in the staging area and the working directory

```
git reset <file>
//Remove the specified file from the staging area, but leave the working directory unchanged. This unstages a file without overwriting any changes

git reset
//Reset the staging area to match the most recent commit, but leave the working directory unchanged. This unstages all files without overwriting any changes, giving you the opportunity to re-build the staged snapshot from scratch

git reset --hard
//Reset the staging area and the working directory to match the most recent commit. In addition to unstaging changes, the --hard flag tells Git to overwrite all changes in the working directory, too. Put another way: this obliterates all uncommitted changes, so make sure you really want to throw away your local developments before using it.

git reset <commit>

git reset --hard <commit>

```
All of the above invocations are used to remove changes from a repository. Without the --hard flag, git reset is a way to clean up a repository by unstaging changes or uncommitting a series of snapshots and re-building them from scratch. The --hard flag comes in handy when an experiment has gone horribly wrong and you need a clean slate to work with

Whereas reverting is designed to safely undo a public commit, git reset is designed to undo local changes. Because of their distinct goals, the two commands are implemented differently: resetting completely removes a changeset, whereas reverting maintains the original changeset and uses a new commit to apply the undo

### Don’t Reset Public History
You should never use git reset <commit> when any snapshots after <commit> have been pushed to a public repository

The point is, make sure that you’re using git reset <commit> on a local experiment that went wrong—not on published changes. If you need to fix a public commit, the git revert command was designed specifically for this purpose.

### Examples
#### Unstaging a File
The git reset command is frequently encountered while preparing the staged snapshot. The next example assumes you have two files called hello.py and main.py that you’ve already added to the repository.
```
// Edit both hello.py and main.py

// Stage everything in the current directory
git add .

// Realize that the changes in hello.py and main.py
//should be committed in different snapshots

// Unstage main.py
git reset main.py

// Commit only hello.py
git commit -m "Make some changes to hello.py"

// Commit main.py in a separate snapshot
git add main.py
git commit -m "Edit main.py"
```
As you can see, git reset helps you keep your commits highly-focused by letting you unstage changes that aren’t related to the next commit.

#### Removing Local Commits
 It demonstrates what happens when you’ve been working on a new experiment for a while, but decide to completely throw it away after committing a few snapshots.

```
// Create a new file called `foo.py` and add some code to it

# Commit it to the project history
git add foo.py
git commit -m "Start developing a crazy feature"

# Edit `foo.py` again and change some other tracked files, too

# Commit another snapshot
git commit -a -m "Continue my crazy feature"

# Decide to scrap the feature and remove the associated commits
git reset --hard HEAD~2

```
The git reset HEAD~2 command moves the current branch backward by two commits, effectively removing the two snapshots we just created from the project history. Remember that this kind of reset should only be used on unpublished commits. Never perform the above operation if you’ve already pushed your commits to a shared repository.

### git clean
removes untracked files from your working directory

The git clean command is often executed in conjunction with git reset --hard. Remember that resetting only affects tracked files, so a separate command is required for cleaning up untracked ones. Combined, these two commands let you return the working directory to the exact state of a particular commit.

```
//Perform a “dry run” of git clean. This will show you which files are going to be removed without actually doing it.
git clean -n

//Remove untracked files from the current directory. The -f (force) flag is required unless the clean.requireForce configuration option is set to false (it's true by default). This will not remove untracked folders or files specified by .gitignore.
git clean -f

git clean -f <path>

//Remove untracked files and untracked directories from the current directory.
git clean -df

//Remove untracked files from the current directory as well as any files that Git usually ignores.
git clean -xf

```

The git reset --hard and git clean -f commands are your best friends after you’ve made some embarrassing developments in your local repository and want to burn the evidence. Running both of them will make your working directory match the most recent commit, giving you a clean slate to work with.

he git clean command can also be useful for cleaning up the working directory after a build. For example, it can easily remove the .o and .exe binaries generated by a C compiler. This is occasionally a necessary step before packaging a project for release. The -x option is particularly convenient for this purpose.

Keep in mind that, along with git reset, git clean is one of the only Git commands that has the potential to permanently delete commits, so be careful with it


```
// Edit some existing files
# Add some new files
# Realize you have no idea what you're doing

# Undo changes in tracked files
git reset --hard

# Remove untracked files
git clean -df
```

## Rewriting history
### git commit --amend
never amend commits that have been pushed to a public repository.
Amended commits are actually entirely new commits, and the previous commit is removed from the project history. This has the same consequences as resetting a public snapshot

The git commit --amend command is a convenient way to fix up the most recent commit. It lets you combine staged changes with the previous commit instead of committing it as an entirely new snapshot. It can also be used to simply edit the previous commit message without changing its snapshot

```
// Edit hello.py and main.py
git add hello.py
git commit

# Realize you forgot to add the changes from main.py
git add main.py
git commit --amend --no-edit
```


Combine the staged changes with the previous commit and replace the previous commit with the resulting snapshot. Running this when there is nothing staged lets you edit the previous commit’s message without altering its snapshot

### git rebase
Don’t Rebase Public History

Rebasing is the process of moving a branch to a new base commit. 

From a content perspective, rebasing really is just moving a branch from one commit to another. But internally, Git accomplishes this by creating new commits and applying them to the specified base—it’s literally rewriting your project history. It’s very important to understand that, even though the branch looks the same, it’s composed of entirely new commits.

`git rebase <base>`
Rebase the current branch onto <base>, which can be any kind of commit reference (an ID, a branch name, a tag, or a relative reference to HEAD).

#### Discussion
The primary reason for rebasing is to maintain a linear project history. For example, consider a situation where the master branch has progressed since you started working on a feature:

```
// Start a new feature
git checkout -b new-feature master
# Edit files
git commit -a -m "Start developing a feature"
```

In the middle of our feature, we realize there’s a security hole in our project

```

// Create a hotfix branch based off of master
git checkout -b hotfix master
# Edit files
git commit -a -m "Fix security hole"
# Merge back into master
git checkout master
git merge hotfix
git branch -d hotfix

```

After merging the hotfix into master, we have a forked project history. Instead of a plain git merge, we’ll integrate the feature branch with a rebase to maintain a linear history:

```
git checkout new-feature
git rebase master

```

This moves new-feature to the tip of master, which lets us do a standard fast-forward merge from master:

```
git checkout master
git merge new-feature
```

### git rebase -i

```
an interactive rebasing session
```

#### Examples
```
// Start a new feature
git checkout -b new-feature master
# Edit files
git commit -a -m "Start developing a feature"
# Edit more files
git commit -a -m "Fix something from the previous commit"

# Add a commit directly to master
git checkout master
# Edit files
git commit -a -m "Fix security hole"

# Begin an interactive rebasing session
git checkout new-feature
git rebase -i master

```

The last command will open an editor populated with the two commits from new-feature, along with some instructions:

```
pick 32618c4 Start developing a feature
pick 62eed47 Fix something from the previous commit
```

You can change the pick commands before each commit to determine how it gets moved during the rebase. In our case, let’s just combine the two commits with a squash command:
```
pick 32618c4 Start developing a feature
squash 62eed47 Fix something from the previous commit

```

Save and close the editor to begin the rebase. This will open another editor asking for the commit message for the combined snapshot. After defining the commit message, the rebase is complete and you should be able to see the squashed commit in your git log output. 

Note that the squashed commit has a different ID than either of the original commits, which tells us that it is indeed a brand new commit.

Finally, you can do a fast-forward merge to integrate the polished feature branch into the main code base:
```
git checkout master
git merge new-feature

```


### git reflog
Git keeps track of updates to the tip of branches using a mechanism called reflog. This allows you to go back to changesets even though they are not referenced by any branch or tag. After rewriting history, the reflog contains information about the old state of branches and allows you to go back to that state if necessary.

```
git reflog

git reflog --relative-date
```
Every time the current HEAD gets updated (by switching branches, pulling in new changes, rewriting history or simply by adding new commits) a new entry will be added to the reflog

```
0a2e358 HEAD@{0}: reset: moving to HEAD~2
0254ea7 HEAD@{1}: checkout: moving from 2.2 to master
c10f740 HEAD@{2}: checkout: moving from master to 2.2
```
The reflog above shows a checkout from master to the 2.2 branch and back. From there, there's a hard reset to an older commit. The latest activity is represented at the top labeled HEAD@{0}.

If it turns out that you accidentially moved back, the reflog will contain the commit master pointed to (0254ea7) before you accidentially dropped 2 commits.

