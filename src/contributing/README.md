# Contributing

## Understanding the structure of the Darling project

### Submodules

The Darling project relies heavily on the usage of submodules. The best way to understand a submodule is to think of it as repo inside of another repo.

The [`.gitmodules` file](https://github.com/darlinghq/darling/blob/master/.gitmodules) provides a list of submodules that the `darling` repo uses.

For example, let's take a look at the `darling-xnu` repo that the `darling` repo includes.

```
[submodule "src/external/xnu"]
path = src/external/xnu
url = ../darling-xnu.git
```

The path indicates the location of the submodule, while the url is the link to the git repo.

> [!NOTE]
> A git submodule's URL path would usually be absolute instead of relative.
> ```
> [submodule "src/external/xnu"]
> path = src/external/xnu
> url = https://github.com/darlinghq/darling-xnu.git
> ```
> However, the Darling team has made the deliberate decision to use a relative path instead, mainly for the following reasons:
> * To allow cloning/updating the project either through `https` or `ssh`
> * To make it easier to hard fork this project.

When you `cd` into the `src/external/xnu` directory, you are working off of the `darling-xnu` repo, instead of the usual `darling` repo.

```bash
cd ~/Downloads/darling
git remote -v
# origin  https://github.com/darlinghq/darling.git (fetch)
# origin  https://github.com/darlinghq/darling.git (push)
```
```bash
cd ~/Downloads/darling/src/external/xnu
git remote -v
# origin  https://github.com/darlinghq/darling-xnu.git (fetch)
# origin  https://github.com/darlinghq/darling-xnu.git (push)
```

[For additional details on git submodules, we recommend reading the submodule section in the git-scm website](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
## Setting up your development environment

If you have not already, [please see instruction page for guidance on how to set up and build Darling.](../build-instructions.md)

We recommend adding the following additional flags when configuring the project with cmake:

```bash
cmake .. -DCMAKE_BUILD_TYPE=Debug -DCOMPONENTS=all
```

## Creating your fork

Unless you are one of the core Darling developers who has write access to the Darling repos, you will need to create a fork to push your changes to. [If you are unsure about how to fork a repository on GitHub, please refer to the following instructions for guidance.](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo#forking-a-repository)

If the changes you want to make live in a submodule (usually located in `src/external`. [See `.gitmodules` file for a list of all of the submodules Darling uses](https://github.com/darlinghq/darling/blob/master/.gitmodules)), you’ll need to `cd` into the submodule’s directory and use git remote to get the URL

```bash
cd src/external/xnu/
git remote get-url origin
```
```
https://github.com/darlinghq/darling-xnu.git
```

Normally, you are expected to create a clone of your fork and commit your changes on that cloned fork. For Darling, we recommend working off the official repo clone you created earlier and add your fork as an additional remote.

# Adding your fork as an additional remote

By default, a fresh clone of Darling will have the following remote:

```bash
git remote -v
```
```
origin  https://github.com/darlinghq/darling.git (fetch)
origin  https://github.com/darlinghq/darling.git (push)
```

Use `git remote add` to add your forked repo, like so:

```bash
git remote add <YOUR GITHUB USERNAME> https://github.com/<YOUR GITHUB USERNAME>/darling.git
```

If you are working off of a submodule, make sure to `cd` into the submodule’s directory first and use the forked submodule URL:

```bash
cd src/external/xnu/
git remote add <YOUR GITHUB USERNAME> https://github.com/<YOUR GITHUB USERNAME>/darling-xnu.git
```

If you did it correctly, you should see the following:

```bash
git remote -v
```
```
<YOUR GITHUB USERNAME>    https://github.com/<YOUR GITHUB USERNAME>/darling.git (fetch)
<YOUR GITHUB USERNAME>    https://github.com/<YOUR GITHUB USERNAME>/darling.git (push)
origin  https://github.com/darlinghq/darling.git (fetch)
origin  https://github.com/darlinghq/darling.git (push)
```

## Pushing your changes to your fork

After you have created a branch and committed your changes, you can push your commits to your fork by using the following command.

```bash
git push -u <YOUR GITHUB USERNAME> <BRANCH NAME>
```

The `-u <YOUR GITHUB USERNAME>` part is only necessary when a branch has never been pushed to your fork before.

## Submit a pull request

On the GitHub page of your fork, select the branch you just pushed from the branch dropdown menu (you may need to reload). Click the *New pull request* button. Give it a useful title and descriptive comment. After this, you can click create.

After this, your changes will be reviewed by a Darling project member.