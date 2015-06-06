---
layout: post
title: "git tutorials -- workflows"
date: 2015-05-29 10:20:33 +0800
comments: true
categories: Git
---
## git tutorials

### Centralized workflow
![](https://www.atlassian.com/git/images/tutorials/collaborating/comparing-workflows/centralized-workflow/01.svg)
```
//Someone initializes the central repository
ssh user@host git init --bare /path/to/repo.git

//Everybody clones the central repository
git clone ssh://user@host/path/to/repo.git

# edit stage commits

//origin is the remote connection to the central repository
git push origin master


//another push
git push origin master

error: failed to push some refs to '/path/to/repo.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Merge the remote changes (e.g. 'git pull')
hint: before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.

git pull --rebase origin master

# Unmerged paths:
# (use "git reset HEAD <some-file>..." to unstage)
# (use "git add/rm <some-file>..." as appropriate to mark resolution)
#
# both modified: <some-file>

# edit the file
git add <some-file> 
git rebase --continue

//you can abort and roll back
//right back to where you started before you ran [git pull --rebase]
git rebase --abort

//or push
git push origin master
```

<!--more-->

### Feature Branch workflow

```
git checkout -b marys-feature master

#Mary edits, stages, and commits changes
git status
git add <some-file>
git commit

//This command pushes marys-feature to the central repository (origin), and the -u flag adds it as a remote tracking branch. After setting up the tracking branch, Mary can call git push without any parameters to push her feature.

git push -u origin marys-feature //first time
git push origin marys-feature    // after

git push

// merge the feature into the stable project 
git checkout master
git pull
git pull origin marys-feature
git push

```
First, whoever’s performing the merge needs to check out their master branch and make sure it’s up to date. Then, git pull origin marys-feature merges the central repository’s copy of marys-feature. You could also use a simple git merge marys-feature, but the command shown above makes sure you’re always pulling the most up-to-date version of the feature branch. Finally, the updated master needs to get pushed back to origin.

### Gitflow Workflow

#### Historical Branches
Instead of a single master branch, this workflow uses two branches to record the history of the project. The master branch stores the official release history, and the develop branch serves as an integration branch for features. It's also convenient to tag all commits in the master branch with a version number.

#### Feature Branches
Each new feature should reside in its own branch, which can be pushed to the central repository for backup/collaboration. But, instead of branching off of master, feature branches use develop as their parent branch. When a feature is complete, it gets merged back into develop. Features should never interact directly with master.


#### Example
```
//1. one developer to create an empty develop branch locally and push it to the server
git branch develop
git push -u origin develop

//2  Other developers should now clone the central repository and create a tracking branch for develop
git clone ssh://user@host/path/to/repo.git
git checkout -b develop origin/develop

// 3 create branche for features develop
git checkout -b some-feature develop

// 4 edit, stage, commit
git status
git add <some-file>
git commit

// 5 makes sure the develop branch is up to date 
git pull origin develop

// 6 merge or rebase
git checkout develop
git merge some-feature
git push
git branch -d some-feature

//7 prepare a release
//clean up the release, test everything, update the documentation, and do any other kind of preparation for the upcoming release
git checkout -b release-0.1 develop

//8 merges it into master and develop, then deletes the release branch
git checkout master
git merge release-0.1
git push
git checkout develop
git merge release-0.1
git push
git branch -d release-0.1


//9 tag the commit for easy reference
git tag -a 0.1 -m "Initial public release" master
git push --tags

//10 creates a maintenance branch off of master, 
//fixes the issue with as many commits as necessary, 
//then merges it directly back into master and develop

git checkout -b issue-#001 master
# Fix the bug
git checkout master
git merge issue-#001
git push

git checkout develop
git merge issue-#001
git push
git branch -d issue-#001


```

### Forking WorkFlow

```
Example
//1. create an official repository on a server
ssh user@host
git init --bare /path/to/repo.git

//2. fork

//3 clone
git clone https://user@bitbucket.org/user/repo.git
git remote add upstream https://bitbucket.org/maintainer/repo
git remote add upstream https://user@bitbucket.org/maintainer/repo.git


//4.  edit, commit branches
git checkout -b some-feature
# Edit some code
git commit -a -m "Add first draft of some feature"

//5. update master
git pull upstream master 

//6. 
git push origin feature-branch


//7. maintainer view a diff of the changes, comment on it, and perform the merge
git fetch https://bitbucket.org/user/repo feature-branch
# Inspect the changes
git checkout master
git merge FETCH_HEAD

//8. merge publish
git push origin master


//9. others update
git pull upstream master
```
