---
layout: post
title: "git tutorials -- Advanced Tips"
date: 2015-05-29 10:06:02 +0800
comments: true
categories: 
---
## git tutorials

### Splitting a Commit
```
git rebase -i HEAD~3

// edit second to edit
pick f7f3f6d changed my name a bit
edit 310154e updated README formatting and added blame
pick a5f4a0d added cat-file

//do a mixed reset of that commit with `git reset HEAD^` effectively undoes that commit and leaves the modified files unstaged
$ git reset HEAD^
$ git add README
$ git commit -m 'updated README formatting'
$ git add lib/simplegit.rb
$ git commit -m 'added blame'
$ git rebase --continue


$ git log -4 --pretty=format:"%h %s"
1c002dd added cat-file
9b29157 added blame
35cfb2b updated README formatting
f3cc40e changed my name a bit

```


### The Nuclear Option: filter-branch
#### Removing a File from Every Commit
```
//remove passwords.txt form all history
$ git filter-branch --tree-filter 'rm -f passwords.txt' HEAD

//remove all accidentally committed editor backup files
git filter-branch --tree-filter 'rm -f *~' HEAD.

//To run filter-branch on all your branches, you can pass --all to the command
$ git filter-branch --tree-filter 'rm -f passwords.txt' HEAD --all

```


### important
1. you should always create a new branch before adding commits to a detached HEAD
2. For this reason, git revert should be used to undo changes on a public branch, and git reset should be reserved for undoing changes on a private branch

<!--more-->