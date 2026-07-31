# slac-tools Quick Start Guide

This page is for first time developers contributing to [slac-tools](https://github.com/slaclab/slac-tools). Contributions are welcome.

## Prerequisites

Before you start, you will need:
- A [GitHub account](https://github.com/join).
- Git installed on your machine. See [GitHub's install guide](https://docs.github.com/en/get-started/git-basics/set-up-git).
- Membership in the [slaclab](https://github.com/slaclab) organization, or a fork of the repository you want to contribute to.
- We also have a [slaclab\hla](https://github.com/orgs/slaclab/teams/hla) team you can join for access to all tools repos. 

## Cloning the repo

Clone the repository to your laptop or a SLAC dev system:

```bash
git clone https://github.com/slaclab/slac-tools.git
cd slac-tools
```

If you do not have write access to slaclab, fork the repository first and clone your fork. See [GitHub's fork documentation](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo).

## Starting a branch from an issue

Please try to develop code on branches that are associated with issues. Find or write an issue related to your changes, then create a branch from the issue page. You can create a branch from an issue on the lower right hand side of the issue page.

<img width="1732" height="938" alt="create-issue-branch" src="https://github.com/user-attachments/assets/99257d5b-52ee-421c-8ea9-495c616f2be5" />

Fetch the new branch locally and switch to it:

```bash
git fetch origin
git checkout <branch-name>
```

## Making changes

Edit the files, then stage and commit:

```bash
git add <files>
git commit -m "Short description of your change"
git push origin <branch-name>
```

## Opening a pull request (PR)

When you are ready to merge your code into the main branch, open a pull request. Linting and test workflows will automatically run on your code. One review and approval is required before merging.

To open a PR, go to the repository on GitHub and follow [GitHub's PR instructions](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request). In the PR description, link the related issue (for example, `Closes #1`) so it closes automatically when the PR is merged.

## Responding to reviews

Reviewers may leave comments or request changes. Push additional commits to the same branch to update the PR. See [GitHub's review documentation](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews).

## Merging

Once your PR has passed checks and received an approval, it can be merged. If you have permission, use the "Squash and merge" option on the PR page. Otherwise, a HLA developer will merge it for you. See [GitHub's merge documentation](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request).

After merging, delete the branch on GitHub and clean up locally:

```bash
git checkout main
git pull origin main
git branch -d <branch-name>
```
You can also delete the branch from the GitHub webpage.