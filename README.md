# Branch Per Environment

Having one branch per environment is a very common branching strategy in Git
repositories. The choice of strategy when merging from one environment branch
to another can have a huge impact on the readability of the Git history. This
repository demonstrates the process for transitioning from a "merge commit"
(`--no-ff`) strategy to a "fast-forward" (`--ff-only`) strategy in an existing
repository. I will not debate the merits of the branch-per-environment Git
workflow, nor of the choice of merge strategy, here.

To be clear, we are only discussing the strategy when merging from one
environment branch to another, not when merging a feature branch into the main
development branch.

When performing a "fast-forward" merge, there is a restriction in Git that the
source branch be strictly ahead of the target branch. But when using the "merge
commit" strategy, the target branch will gain a new commit each time that is
not in the source branch (the merge commit itself). This can make it difficult
to transition from the "merge commit" strategy to the "fast-forward" strategy,
because from the starting point there will always be commits in the higher
environment branches which are not in the lower environment branches, making
"fast-forward" merges initially impossible. The process needs to address this
issue.

Our process will have the following two constraints:

1. We will not perform any action on a long-lived branch that would rewrite the
  history of the branch, such as performing a `rebase` or a `reset`.
2. The process must be able to accommodate changes which are committed to lower
  environment branches while the process is unfolding. That is, the process
  can't assume that all work will stop while it plays out.

But first, we will define what is meant by the two merge strategies.


## Merge Commit Strategy

The "merge commit" strategy incorporates the changes from the source branch
into the target branch by creating a merge commit. In GitHub, this is the 
[default merge strategy][1], and it's well supported in [Azure Repos][2] and
Bitbucket as well. In Git, this can be achieved by running the following:

```
git checkout target
git merge source --no-ff
```

The "merge commit" strategy typically produces Git logs which look like the
following:

```
*   ####### (HEAD -> prod) Release Feature 2 to prod (#7)
|\
| | *   ####### (develop) Feature 3 (#5)
| | |\
| * | | ####### (test) Release Feature 2 to test (#6)
| |\| |
| | | * ####### Feature 3
| | |/
* | | ####### Release Feature 1 to prod (#4)
|\| |
| | *   ####### Feature 2 (#3)
| | |\
| | | * ####### Feature 2
| | |/
| * / ####### Release Feature 1 to test (#2)
|/| |
| |/
| * ####### Feature 1 (#1)
|/|
| * ####### Feature 1
|/
* ####### Initial commit
```


## Fast-forward Strategy

The "fast-forward" strategy incorporates changes from the source branch into
the target branch by using a fast-forward merge. In GitHub, this can only be
achieved by manually doing a fast-forward merge from your own repository and
pushing to GitHub, which will automatically close the relevant pull request. In
[Azure Repos][2], this can be achieved via the "Rebase and fast-forward"
option, and in Bitbucket via the ["Fast-forward" option][3]. In Git, this can
be achieved by running the following:

```
git checkout target
git merge source --ff-only
```

The "fast-forward" strategy typically produces Git logs which look like the
following:

```
*   ####### (develop) Feature 5 (#12)
|\
| * ####### Feature 5
|/
*   ####### (HEAD -> prod, test) Feature 4 (#10)
|\
| * ####### Feature 4
|/
*   ####### Continue switch to fast-forward merges (#9)
```


## Branch Structure

This repository will have three long-lived environment branches, `develop`,
`test` and `prod`. The workflow for releasing code is as follows:

1. Create a feature branch, e.g. `feature/1`, from the `develop` branch
2. Merge the feature branch into the `develop` branch. All commits to the
  `develop` branch will be deployed to the Development environment. In this
  repository, we will always use the "merge commit" strategy, but this process
  works fine if your team prefers to use the "squash and merge" or the "rebase
  and merge" strategies. This is a personal decision for your team, and I will
  not debate the merits of these strategies here.
3. Merge the `develop` branch into the `test` branch. All commits to the `test`
  branch will be deployed to the Test environment. This is usually only done
  after a handful of changes have been made in the Development environment
  which are ready to be tested formally.
4. Merge the `test` branch into the `prod` branch. All commits to the `prod`
  branch will be deployed to the Production environment. This is usuallyy only
  done after a handful of changes have been made in the Test environment which
  are ready to be released.

When performing merges for steps 3 and 4, we will initially use the "merge
commit" strategy, and then later we will begin the process to transition to the
"fast-forward" strategy.

In order to demonstrate that work doesn't stop while you make this transition,
we will continue to push changes to lower environments as we make the
transition, and the process can accommodate these changes gracefully.


## Setup

We begin setting up this repository by creating the `README.md` file as the
initial commit on the `develop` branch, and then creating the `test` and `prod`
branches.

```
git init
git checkout -b develop
echo "Initial commit" > CHANGELOG.md
git add CHANGELOG.md
git commit -m"Initial commit"
git branch test
git branch prod
```

At this stage, the Git commit history looks like this

```
* ####### (HEAD -> develop, test, prod) Initial commit
```


## Developing Features

New features will be added through a typical pull request-based workflow. We
create a feature branch with an auto-incrementing feature ID, starting with
`feature/1`, and create a single commit. For feature branches, we will always
use the "merge commit" merge strategy, and we will always delete our feature
branches once the corresponding pull request has been merged.

```
git checkout -b feature/1 develop
echo "Feature 1" >> CHANGELOG.md
git add CHANGELOG.md
git commit -m"Feature 1"
git checkout develop
git merge --no-ff feature/1 -m"Feature 1 (#1)"
git branch -d feature/1
```

At this stage, the Git commit history looks like this

```
*   ####### (HEAD -> develop) Feature 1 (#1)
|\
| * ####### Feature 1
|/
* ####### (test, prod) Initial commit
```


[1]: https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-request-merges
[2]: https://docs.microsoft.com/en-au/azure/devops/repos/git/pull-requests?view=azure-devops#complete-the-pull-request
[3]: https://bitbucket.org/blog/fast-forward-merges-bitbucket-cloud-default-like
