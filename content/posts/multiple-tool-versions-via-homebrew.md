---
title: "Multiple Tool Versions via Homebrew"
date: 2020-04-12T19:41:08+02:00
draft: false
categories:
- Tips & Tricks
tags:
- hugo
- homebrew
- git
- cli
- mac
---

Recently, I started using [Hugo](https://gohugo.io/) with an older project.
After I pulled the code and ran `hugo`, it failed.
It turned out it was not yet compatible with the latest version that got automatically installed via [Homebrew]([https://](https://brew.sh/)).

Well, no big deal, Homebrew allows for installing older versions of formulae.
So I got it working.

Shortly after that, I started another new project (this blog, BTW).
Guess what happened? It broke again, this time because the installed version of `hugo` was too old and not compatible with the configured theme.

Well, that is a bit of a problem, but there is hope:

## Installing an old Version

First of all, let's see how to install the old version in the first place.

The nice thing is that Homebrew entirely relies on files from a [public repository on GitHub](https://github.com/Homebrew/homebrew-core).
The bad thing is that **all** default formulae come from this repository, and we have to inspect the commit history to find the desired version of a tool. This is a task that is most likely going to overwhelm the GitHub UI at some point.

### Get the Last Versions from Homebrew

Each formula is `*.rb` file in the `Formula` directory of the Git repository and each commit contains information about a tool and version.

Hence, we only need to find the respective commit for what we want to install.
For that, we can use either Homebrew itself or Git commands.

#### Using Homebrew

Cloning and pulling the Homebrew repository just to inspect the history might seem a bit too much,
and luckily has us covered:

```sh
$ brew log --oneline hugo
```

This will output the last commits for the formula.
However, the installation of Homebrew relies on a shallow clone of the corresponding Git repository, which means there is only partial data available, i.e., the desired version we want to install might not be included in the result anymore.

If we are lucky, and the version is included in the output, we can proceed with the next step: [Find the Formula]({{< relref "#find-the-formula" >}}).
Otherwise, we have to use Git to access the full history.

#### Using Git

In order to be able to have a look at the full history from the command line, we have to clone the repository,
even though we only need to retrieve the commit history, i.e.:

- We don't need any actual files.
- We don't need an actual checkout.
- We're only interested in the `master` branch.

The following command gives us exactly that:

```sh
$ git clone \
  --filter=blob:none \
  --no-checkout \
  --single-branch \
  --branch master \
  https://github.com/Homebrew/homebrew-core.git
```

Although we did not check out any branch, we still received more than 260 MB of commit history data.
Obviously, we would like to avoid that, if possible.

After cloning the repository (history information), we can check the logs for commits related to the tool and version we are interested in, `hugo` and version `0.41` in this example:

```
$ git log --oneline | grep hugo | grep -F 0.41
```

1. Get the commit history in one-line format
2. Filter all commits related to `hugo`
3. Filter all commits related to version `0.41` (using a fixed string, otherwise the `.` had to be escaped.

This returns the following result:

```
b1e187384b hugo: update 0.41 bottle.
824a875022 hugo 0.41
```

We are only interested in the most recent commit for the version.

### Find the Formula

With the commit ID, we can identify the file required for the installation of the version at that time, if necessary:

```sh
$ git diff-tree --no-commit-id --name-only -r b1e187384b
Formula/hugo.rb
```

Usually, this result should not be a big surprise since all formulae are located in the `Formula` directory and the information about a formula also cover this:

```sh
$ brew info hugo
hugo: stable 0.69.0 (bottled), HEAD
Configurable static site generator
https://gohugo.io/
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/hugo.rb
```

### Install from File Path via Homebrew

Now, we can use the retrieved information to install that the formula at a specific version via Homebrew,
regardless of how old it actually is:

```sh
$ brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/824a875022/Formula/hugo.rb
```

Bringing it all together with `${FORMULA}` and `${VERSION}` representing what we want to install,
we can use the following snippet to install a formula based on the Git commit history:

```sh
$ COMMIT_ID=$(git log --oneline | grep "${FORMULA}" | grep -F "${VERSION}" | awk 'FNR==1 { print $1 }')
$ FILE=$(git diff-tree --no-commit-id --name-only -r ${COMMIT_ID})
$ brew install "https://raw.githubusercontent.com/Homebrew/homebrew-core/${COMMIT_ID}/${FILE}"
```

## Prevent old Version from Upgrades

If you only want to ensure that the installed old version does not get overridden by automatic upgrades that get installed via `brew upgrade`, you can "pin" this version:

```sh
$ brew pin hugo
```

This can be undone with the below command:

```sh
$ brew unpin hugo
```

## Install latest Version

After we have successfully installed the outdated version of our tool,
let's have look at how to install the latest version without upgrading, i.e., overriding the previous version.

### Unlink the installed Version

By unlinking the previously installed version, we simulate the state without the tool being installed:

```sh
$ sh brew unlink hugo
```

Now, it will not be possible to use the command:

```sh
$ hugo version
zsh: command not found: hugo
```

### Install new Version

To install the latest version, first, get the latest formulae and then install it using the following command:

```sh
$ brew update
$ brew install hugo
```

If we run the `version` command again, we will get the expected output:

```sh
$ hugo version
Hugo Static Site Generator v0.69.0/extended darwin/amd64 BuildDate: unknown
```

## Switch between Installed Versions

Now that we have multiple versions of our desired tool available, it is possible to switch between them whenever needed.

### Check available Versions

To check which versions are actually available, use this command:

```sh
$ brew list --versions hugo
```

### Switch to another Version

Assuming that you are still using the latest version that was last installed, we can use the following command to switch back to the outdated version used before that:

```sh
$ brew switch hugo 0.41
```

That's all!

Internally, Homebrew is updating symlinks to point to the respective version we are specifying in the `switch` command.
