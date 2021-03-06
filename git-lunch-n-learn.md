# Git Talk :)

# Git concepts
[Git Concepts](http://git-scm.com/book/en/v2/Getting-Started-Git-Basics)
The charts (snapshots instead of deltas and the workspace, staging index, repo) on this page are pretty helpful.

[Visual Git cheatsheet](http://ndpsoftware.com/git-cheatsheet.html)
Another nice illustration of how changes flow from workspace to staging index to local repo to remote repo.

[SVN commands and their Git equivalent](http://git-scm.com/course/svn.html) Helpful for transitioning from SVN to git.


# Initial Setup

Clone the entire repo `git clone git@github.comcast.com:xbo/deviceDataService.git`

Take a look at the branches`git branch -a`

# Day to day workflow

1. Get latest from remote repo (origin)
2. Create a new branch based off `dev` for your feature work
3. Edit files
4. Take a look at local changes
5. Prepare for a commit by adding files to git's staging area (aka index)
6. Review what you're about to commit
7. Commit
8. Bring your branch up to date by merging the `dev` branch into it
9. Push your branch up to remote repo
10. GOTO 1

### 1. Get latest from remote repo

Switch to your dev branch `git checkout dev`

Fetch from the remote branch and merge it into your local `git pull`

(Another way to do it but without a merge
`git fetch && git reset --hard origin/dev` )

### 2. Create a new branch for your feature/work
Create a new branch based off your current branch (dev)
`git checkout -b update-readme`

### 3. Edit files
`echo "Hello from git" >> README.md`

### 4. Take a look at changes
`git status`
`git diff`

### 5. Prep for commit by adding files to git's staging area
`git add -A`

### 6. Review what you're about to commit
`git diff --staged`

### 7. Commit your changes to your local branch
`git commit -m "Added a basci REAMDE" -m "You can put a longer description in here"`

`git status`

### Amend the last commit (this is only a good idea BEFORE you've pushed your branch)
`git commit --amend`

### 8. Bring your branch up to date by merging the dev branch into it
* Get latest `git fetch`
* Merge dev in: `git merge origin/dev`
* Identify merge conflicts: `git status`
* Fix merge conflicts
* `git status` will walk you through it w/instructions
* `git add conflicted-files`
* `git commit`

### 9. Push the branch for review
`git push origin updated-readme`

### 10. GOTO 1

# Other Stuff

### Merging branches
`git merge --no-ff updated-readme`
I suggest using no-ff b/c it makes it easier to revert merges

### Show commits 
`git log`

```text
commit 061167d8a832e2396a0fa88fa0f4f1a9d795f7cb
Author: Mike Evans <michael_evans@comcast.com>
Date:   Sun Feb 1 07:36:34 2015 -0500

    Basic couchbase doc lookup
```

Show the diffs along with the commits: `git log -p`

### Cherry-picking commits from other branches

Sometimes you just want to pull a single commit from another branch.

* Use `git log` to find the SHA of the commit you want to pull
* `git cherry-pick SHA_GOES_HERE` 

### Pausing your work

Working on a feature branch, not quite ready for a commit, but gotta check on something higher priority
* `git stash` saves your work into the stash queue
* `git checkout higher-priority`
* `git checkout -` switches back to your old branch
* `git stash pop` pop your work off the queue back into your working dir

### Only add bits of files
`git add -p` "stage this hunk" will walk through each change **within** files

### Finding where bugs where introduced

* git bisect start
* git bisect bad                 # Current version is bad
* git bisect good 43de500850b0357f5187e612a97ea05b65defce0


### What's happening behind the scenes
`git reflog`


# Great resources:
[Visual Git Cheatsheet](http://ndpsoftware.com/git-cheatsheet.html)

[Pro Git, great free online book on Git](http://git-scm.com/book/en/v2)
