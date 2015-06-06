---
layout: post
title: "git tutorials -- Collaborating"
date: 2015-05-29 10:05:45 +0800
comments: true
categories: Git
---
## git tutorials
### git remote

```
//List the remote connections you have to other repositories
git remote

//List the remote connections you have to other repositories, include the URL
git remote -v

git remote add <name> <url>
git remote rm <name>
git remote rename <old-name> <new-name>

```

#### The origin Remote
When you clone a repository with git clone, it automatically creates a remote connection called origin pointing back to the cloned repository. This is useful for developers creating a local copy of a central repository, since it provides an easy way to pull upstream changes or publish local commits. This behavior is also why most Git-based projects call their central repository origin.

<!--more-->

### git fetch
The git fetch command imports commits from a remote repository into your local repo. The resulting commits are stored as remote branches instead of the normal local branches that we’ve been working with. This gives you a chance to review changes before integrating them into your copy of the project.


//Fetch all of the branches from the repository. This also downloads all of the required commits and files from the other repository.
`git fetch <remote>`

git fetch <remote> <branch>

#### Discussion
Fetching is what you do when you want to see what everybody else has been working on. Since fetched content is represented as a remote branch, it has absolutely no effect on your local development work. This makes fetching a safe way to review commits before integrating them with your local repository. It’s similar to svn update in that it lets you see how the central history has progressed, but it doesn’t force you to actually merge the changes into your repository.


#### Remote Branches
Remote branches are just like local branches, except they represent commits from somebody else’s repository. You can check out a remote branch just like a local one, but this puts you in a detached HEAD state (just like checking out an old commit). You can think of them as read-only branches. To view your remote branches, simply pass the -r flag to the git branch command. Remote branches are prefixed by the remote they belong to so that you don’t mix them up with local branches. For example, the next code snippet shows the branches you might see after fetching from the origin remote:

```
git branch -r
# origin/master
# origin/develop
# origin/some-feature
```
Again, you can inspect these branches with the usual git checkout and git log commands. If you approve the changes a remote branch contains, you can merge it into a local branch with a normal git merge. So, unlike SVN, synchronizing your local repository with a remote repository is actually a two-step process: fetch, then merge. The git pull command is a convenient shortcut for this process.

#### Examples
This example walks through the typical workflow for synchronizing your local repository with the central repository's master branch.

`git fetch origin`

This will display the branches that were downloaded:

```
a1e8fb5..45e66a4 master -> origin/master
a1e8fb5..9e8ab1c develop -> origin/develop
* [new branch] some-feature -> origin/some-feature
```
The commits from these new remote branches are shown as squares instead of circles in the diagram below. As you can see, git fetch gives you access to the entire branch structure of another repository.

To see what commits have been added to the upstream master, you can run a git log using origin/master as a filter

`git log --oneline master..origin/master`

To approve the changes and merge them into your local master branch with the following commands:
```
git checkout master
git log origin/master
```

Then we can use git merge origin/master
```
git merge origin/master
```
The origin/master and master branches now point to the same commit, and you are synchronized with the upstream developments.


### git pull
Merging upstream changes into your local repository is a common task in Git-based collaboration workflows. We already know how to do this with git fetch followed by git merge, but git pull rolls this into a single command.

`git pull <remote>`
Fetch the specified remote’s copy of the current branch and immediately merge it into the local copy. This is the same as git fetch <remote> followed by git merge origin/<current-branch>.

`git pull --rebase <remote>`
instead of using git merge to integrate the remote branch with the local one, use git rebase

#### Pulling via Rebase
The --rebase option can be used to ensure a linear history by preventing unnecessary merge commits. Many developers prefer rebasing over merging, since it’s like saying, “I want to put my changes on top of what everybody else has done.” In this sense, using git pull with the --rebase flag is even more like svn update than a plain git pull.

`git config --global branch.autosetuprebase always`
After running that command, all git pull commands will integrate via git rebase instead of git merge.

#### Examples
```
git checkout master
git pull --rebase origin
```


### git push
`git push <remote> <branch>`
Push the specified branch to <remote>, along with all of the necessary commits and internal objects. This creates a local branch in the destination repository

`git push <remote> --force`

`git push <remote> --all`
Push all of your local branches to the specified remote.

`git push <remote> --tags`
ags are not automatically pushed when you push a branch or use the --all option. The --tags flag sends all of your local tags to the remote repository.

`git push origin master`

#### Examples
First, it makes sure your local master is up-to-date by fetching the central repository’s copy and rebasing your changes on top of them. The interactive rebase is also a good opportunity to clean up your commits before sharing them. Then, the git push command sends all of the commits on your local master to the central repository
```
git checkout master
git fetch origin master
git rebase -i origin/master
# Squash commits, fix up commit messages etc.
git push origin master
```



### Work flow
1. fork offical repository
2. git clone https://user@bitbucket.org/user/repo.git
3. git checkout -b some-feature
4. # Edit some code
5. git commit -a -m "Add first draft of some feature"

Mary can use as many commits as she needs to create the feature. And, if the feature’s history is messier than she would like, she can use an interactive rebase to remove or squash unnecessary commits. For larger projects, cleaning up a feature’s history makes it much easier for the project maintainer to see what’s going on in the pull request

After her feature is complete, Mary pushes the feature branch to her own Bitbucket repository (not the official repository) with a simple git push:
6. git push origin some-branch

To correct the error, Mary adds another commit to her feature branch and pushes it to her Bitbucket repository, just like she did the first time around. This commit is automatically added to the original pull request, and John can review the changes again, right next to his original comment.


### Using Branches
### git branch
The git branch command lets you create, list, rename, and delete branches. It doesn’t let you switch between branches or put a forked history back together again. For this reason, git branch is tightly integrated with the git checkout and git merge commands.

List all of the branches in your repository.
`git branch`

Create a new branch called <branch>. This does not check out the new branch.
`git branch <branch>`

Delete the specified branch. This is a “safe” operation in that Git prevents you from deleting the branch if it has unmerged changes.
`git branch -d <branch>`

Force delete the specified branch, even if it has unmerged changes. This is the command to use if you want to permanently throw away all of the commits associated with a particular line of development.
`git branch -D <branch>`

Rename the current branch to <branch>.
`git branch -m <branch>`

In Git, branches are a part of your everyday development process. When you want to add a new feature or fix a bug—no matter how big or how small—you spawn a new branch to encapsulate your changes. This makes sure that unstable code is never committed to the main code base, and it gives you the chance to clean up your feature’s history before merging it into the main branch.

#### Example
It's important to understand that branches are just pointers to commits. When you create a branch, all Git needs to do is create a new pointer—it doesn’t change the repository in any other way. So, if you start with a repository that looks like this:

`git branch crazy-experiment`
The repository history remains unchanged. All you get is a new pointer to the current commit:

Note that this only creates the new branch. To start adding commits to it, you need to select it with git checkout, and then use the standard git add and git commit commands. Please see the git checkout section of this module for more information.

`git branch -d crazy-experiment`

`git checkout <existing-branch>`


Create and check out <new-branch>.
`git checkout -b <new-branch>`

base the new branch off of <existing-branch> instead of the current branch
`git checkout -b <new-branch> <existing-branch>`


Remember that the HEAD is Git’s way of referring to the current snapshot. the git checkout command simply updates the HEAD to point to either the specified branch or commit. When it points to a branch, Git doesn't complain, but when you check out a commit, it switches into a “detached HEAD” state.

The point is, your development should always take place on a branch—never on a detached HEAD. This makes sure you always have a reference to your new commits. However, if you’re just looking at an old commit, it doesn’t really matter if you’re in a detached HEAD state or not.

####  Example
```
git branch new-feature
git checkout new-feature

# Edit some files
git add <file>
git commit -m "Started work on a new feature"
# Repeat
```

All of these are recorded in new-feature, which is completely isolated from master. You can add as many commits here as necessary without worrying about what’s going on in the rest of your branches. When it’s time to get back to “official” code base, simply check out the master branch:
`git checkout master`

### git merge
Note that all of the commands presented below merge into the current branch. The current branch will be updated to reflect the merge, but the target branch will be completely unaffected. Again, this means that git merge is often used in conjunction with git checkout for selecting the current branch and git branch -d for deleting the obsolete target branch.


Merge the specified branch into the current branch. Git will determine the merge algorithm automatically
`git merge <branch>`


Merge the specified branch into the current branch, but always generate a merge commit (even if it was a fast-forward merge). This is useful for documenting all merges that occur in your repository.
`git merge --no-ff <branch>`

#### fast-forward merge
A fast-forward merge can occur when there is a linear path from the current branch tip to the target branch. Instead of “actually” merging the branches, all Git has to do to integrate the histories is move (i.e., “fast forward”) the current branch tip up to the target branch tip. This effectively combines the histories, since all of the commits reachable from the target branch are now available through the current one. For example, a fast forward merge of some-feature into master would look something like the following:


However, a fast-forward merge is not possible if the branches have diverged. When there is not a linear path to the target branch, Git has no choice but to combine them via a 3-way merge. 3-way merges use a dedicated commit to tie together the two histories. The nomenclature comes from the fact that Git uses three commits to generate the merge commit: the two branch tips and their common ancestor.

### Resolving Conflicts
f the two branches you‘re trying to merge both changed the same part of the same file, Git won’t be able to figure out which version to use. When such a situation occurs, it stops right before the merge commit so that you can resolve the conflicts manually.

The great part of Git's merging process is that it uses the familiar edit/stage/commit workflow to resolve merge conflicts. When you encounter a merge conflict, running the git status command shows you which files need to be resolved. For example, if both branches modified the same section of hello.py, you would see something like the following:
```
// On branch master
// Unmerged paths:
// (use "git add/rm ..." as appropriate to mark resolution)
//
// both modified: hello.py
//

```
Then, you can go in and fix up the merge to your liking. When you're ready to finish the merge, all you have to do is run git add on the conflicted file(s) to tell Git they're resolved. Then, you run a normal git commit to generate the merge commit. It’s the exact same process as committing an ordinary snapshot, which means it’s easy for normal developers to manage their own merges.

Note that merge conflicts will only occur in the event of a 3-way merge. It’s not possible to have conflicting changes in a fast-forward merge.

#### Example
Fast-Forward Merge
```
// Start a new feature
git checkout -b new-feature master

# Edit some files
git add <file>
git commit -m "Start a feature"

# Edit some files
git add <file>
git commit -m "Finish a feature"

# Merge in the new-feature branch
git checkout master
git merge new-feature
git branch -d new-feature
```

3-Way Merge

```

// Start a new feature
git checkout -b new-feature master

# Edit some files
git add <file>
git commit -m "Start a feature"

# Edit some files
git add <file>
git commit -m "Finish a feature"

# Develop the master branch
git checkout master

# Edit some files
git add <file>
git commit -m "Make some super-stable changes to master"

# Merge in the new-feature branch
git merge new-feature
git branch -d new-feature

```
