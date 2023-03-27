+++
title = "Efficient command line git usage"
date = 2023-03-02

[taxonomies]
tags = ["git"]
+++

Here are some command line git tricks that I have come up with over the years to improve my efficiency. I will mostly focus on vanilla git commands because that is the interesting stuff, but it goes without saying that [Github CLI](https://github.com/cli/cli) is a good productivity boost as well.
Unfortunately it only works with Github.

You can also find all of the below snippets in my public [dotfiles repo](https://github.com/vimpostor/dotfiles).

# Rebasing automatically

Often while your pull request is in the review pipeline, you are required to rebase on top of the latest upstream changes.
Of course I needed to automate this with the following zsh function, so now I just need to type `greb` (not to be confused with `grep`) and it will automatically pull the last changes for the default upstream branch, rebase on top of that and print a nice overview of the commits and a shortdiff of what changed upstream since the last branch-off point.

```bash
function greb() {
	UPSTREAM="$(git remote | grep upstream || git remote | grep origin)"
	BRANCH="$UPSTREAM/$(git branch -rl \*/HEAD | head -1 | rev | cut -d/ -f1 | rev)"
	git fetch "$UPSTREAM" && \
	git --no-pager log --reverse --pretty=tformat:%s "$(git merge-base HEAD "$BRANCH")".."$BRANCH" && \
	git rebase "$BRANCH"
}
```

The above assumes that you named the upstream git remote `upstream`, otherwise it will fallback to `origin` which as a nice side effect allows it to work for repos where you are your own "upstream".

The name of the upstream default branch is figured out automatically, so it works for any arbitrary repo as long as the remote is appropriately named.

# Checking out PRs

Most people obviously already know that they don't need to add the fork repo as a new remote to checkout the PR branch, because a new branch is automatically created on the upstream repo.

However even checking out that branch is a flawed solution for multiple reasons. First of all it will pollute your local clone because it will permanently create that branch until you manually delete it. That's just an annoyance though. The real problem is that the branch will cause you trouble later on, when the PR author force-pushes and you want to pull the branch again.

The better solution is to not checkout the branch at all but to fetch the branch and use the largely unknown `FETCH_HEAD` git feature. Not even the [Github CLI](https://github.com/cli/cli) tool uses this feature.
It's a temporary reference that always points to the last fetched content. Hence we can use it to checkout something temporarily.

I have implemented this in the following `gcpr` function, that you can call for example as `gcpr 58` to check out PR #58. This is completely immune to force-pushes as `FETCH_HEAD` doesn't care about that. And as a bonus it also doesn't pollute your local branch list.

```bash
# Takes the PR number as only argument
function gcpr() {
	git fetch -q origin pull/"$*"/head 2>/dev/null || \
	git fetch -q upstream pull/"$*"/head 2>/dev/null || \
	git fetch -q origin merge-requests/"$*"/head 2>/dev/null || \
	git fetch -q upstream merge-requests/"$*"/head && git checkout FETCH_HEAD
}
```

Note that the duplicate lines are needed to be compatible with both Github and Gitlab.

# Better diffs

We all know that the default `git diff` view sucks big time.
Naturally many people think that to enjoy sane diffs, they need to install some bloated tool like [diff-so-fancy](https://github.com/so-fancy/diff-so-fancy).
But the solution is much easier.

In fact git ships with a sane diff tool called `diff-highlight` capable of all that fancy stuff like inter-line diffs, it's just not used by default.
You can enable it with the following command, however note that the exact path may vary based on your distro.

```bash
git config --global core.pager '/usr/share/git/diff-highlight/diff-highlight| less'
```

There also is [delta](https://github.com/dandavison/delta) with even more features such as syntax highlighting, but for me personally that doesn't really help with the actual diff.

# Automatic fixup

If you think this section is just about `git rebase --autosquash`, you won't be prepared for the magic that is [git-absorb](https://github.com/tummychow/git-absorb). `git absorb` is a port of the awesome [absorb feature from mercurial](https://www.mercurial-scm.org/repo/hg/rev/5111d11b8719).

Imagine you want to merge a branch of multiple commits into `master`. However, the maintainer has requested some changes and you have implemented them and staged them. Obviously you want to preserve a clean git history, so you need to squash those changes into the already existing commits.

Now you can either go through the cumbersome process of manually finding out the commits that those changes belong to and squash them. Or you can simply run `git absorb --and-rebase`. It's like dark magic! It automatically finds out the correct commits to squash into by checking for which patches commute. That's it, you only need to confirm the rebase and you are done.

# Safe force pushing

When one needs to force-push a rebased branch, most people will tend to use `git push --force`. Usually this is fine as you are hopefully not force pushing to a shared branch anyway.

However there is a much better alternative, that also works safely on shared branches. With the `--force-with-lease` and `--force-if-includes` flags, git will perform some sanity checks before actually going through with the forced push.
I have setup an alias for this:

```bash
alias gpf='git push --force-with-lease --force-if-includes'
```

With this in place, the push will fail if a colleague pushed another patch that you weren't aware of to the branch. With `--force` you would have overwritten that patch and never realized it!

Instead with these flags you will first have to fetch and see the remote changes, hopefully sparing you the anger of your colleague.

# Quick amends

Sometimes I have already made a commit, but realize I need to amend a simple change to the same commit. Obviously I don't want to retype the commit message, so I made this simple alias that amends all changes into the last commit:

```bash
alias gcea='git commit --amend --no-edit -a'
```

If you only want to amend staged changes, feel free to drop the `-a`.

# Fully automated git bisect

When a regression is introduced into a codebase, using a debugger to find the source of the problem might be a bit overkill.
Instead it is often much more practical to use `git bisect` to find the commit that introduced the regression in the first place, especially if you are unfamiliar with the codebase. With any luck, the context of the offending commit provides enough information to fix the bug, avoiding a dreaded session with your _debugging duck_ of choice altogether!

Since `git bisect` uses binary search, it finds the introduction of the regression reasonably fast in logarithmic time complexity.
However manually compiling, testing and entering `git bisect good` or `git bisect bad` for every revision gets boring very quickly.

The true magic unfolds when you write a script that does all of this automatically. You start it with `git bisect run`, go grab a cup of coffee for a couple minutes, and when you're back, git has finished and already presents the problematic commit to you, no further actions required.

In the example below I bisect a regression in [vim](https://github.com/vim/vim)'s codebase. The automatic test is already written and exits with code `0` if it passes and if it fails it returns with a non-zero code. This is how git knows whether a revision is good or bad.
Then you can just pass the script to `git bisect run` and leave it running for a while.

Now enjoy this automatic bisection over a range of 229 commits, finding the bad commit within only 2 minutes.

{{ asciinema(id=570941) }}

In this case it was less of a regression and rather a slight change in behaviour, that I was able to work around downstream instead.

Sometimes you don't know if a revision is good or bad. This can be the case when there is a temporary compilation issue over a range of commits for example. In that case you achieve the equivalent of `git bisect skip` by returning with the special exit code `125`.
