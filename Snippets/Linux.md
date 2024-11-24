
#### Linux processes
```shell
ps -auxwf
```

#### Git sign-off

##### Fix Option #1: Amend Single Commit

If there is only a single commit in your pull request, adding the signature is as simple as:

```shell
git commit --amend --no-edit --signoff
git push -f origin <your branch here, probably "master">
```

You can always squash several commits together; see: [StackOverflow](https://stackoverflow.com/questions/5189560/squash-my-last-x-commits-together-using-git).

`-s` is a shortcut for `--signoff` - you can use it instead.

##### Fix Option #2: rebase ([git>=2.13](https://github.com/git/git/blob/master/Documentation/RelNotes/2.13.0.txt#L189))

If there is more than one commit in your pull request and your git client is modern enough (2.13+), rebase the required number of commits with `--signoff`:

```shell
git rebase --signoff HEAD^^
```

Write `^` as many times as there are commits in your pull request. Then, force push:

```shell
git push -f origin <your branch here, probably "master">
```

#### Upstream

Fork and clone repository:

```bash
git@bitbucket.org:my-user/some-project.git
# Verify upstream
git remote -v
origin  git@bitbucket.org:my-user/some-project.git (fetch) 
origin  git@bitbucket.org:my-user/some-project.git (push)
# Add upstream
git remote add upstream git@bitbucket.org:some-gatekeeper-maintainer/some-project.git
git remote -v
origin git@bitbucket.org:my-user/some-project.git (fetch) 
origin git@bitbucket.org:my-user/some-project.git (push) 
upstream git@bitbucket.org:some-gatekeeper-maintainer/some-project.git (fetch)
upstream  git@bitbucket.org:some-gatekeeper-maintainer/some-project.git (push)
git fetch upstream
git checkout main 
git merge upstream/main
git checkout -b feature-x 
# some work and some commits happen 
# some time passes
git fetch upstream 
git rebase upstream/main
```

