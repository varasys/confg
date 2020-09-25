# confg - config dotfile goodness with git

`confg` (as in CONFig Git) is a posix shell script to manage configuration dotfiles in git repositories.

## Background

So, for example, you have just finished updating your .vimrc file (or any other configuration file that is important to you) and it is perfect. So you want to save it somewhere and it would be nice if there was some sort of version control of it, and as an added bonus be able to get to it easily if you need it on another computer.

There have been several article about how to do this with `git` floating around the internet. The one that this work is inspired from is [here](https://www.atlassian.com/git/tutorials/dotfiles).

The simple summary is to create a bare repo somewhere in your $HOME dir, and then use some of git's command line arguments to set the working directory to $HOME and the repo to wherever you created the bare repo dir. Under this scheme, to add your .vimrc file you would use the command `git --git-dir="${REPO}" --work-dir="${HOME}" add ~/.vimrc`. Everything works just like normal git (adding remotes, pushing, pulling, etc.) but with the '--git-dir' and '--work-dir' arguments it will always use there directories.

In order to save keystrokes, the article mentioned above suggests using an alias in your .bashrc (or .zshrc) file like `alias config='/usr/bin/git --git-dir="$HOME/config.git" --work-tree="$HOME"`. Then, for example, you could add a remote repo to push to with `config remote add <name> <url>`. And then actually push to it with `config push <name>`.

But some of my config files have secrets (ie. my '.password-store' directory) which I wouldn't want to push to github (for example), and I also have some system wide configuration that I want to manage this way which are not in my $HOME directory. 

So this script applies the same concepts, but uses some symlinks to provide four commands:

* `confg`: (CONFig Git) manages configuration files strictly in the user $HOME dir that **DO NOT** contain sensitive information such as passwords
* `seconfg`: (SEcure CONFig Git) manages configuration files strictly in the user $HOME dir that **DO** contain sensitive information such as passwords
* `syconfg`: (SYstem CONFig Git) manages system wide configuration files located anywhere under the '/' directory
* `aconfg`: (All CONFig Git) sequentially run each of the previous three commands with the same arguments (ie. `aconfg push` will push all three repos to the default remote configured for each of them)

Note that since the `syconfg` variation is not user specific the script will call `sudo` if you are not already root.

There is only one script, but by symlinking to the single script with different names for each symlink, the script can inspect what it was called as and act appropriately.

The script will automatically create the local bare git repos as needed. See the Environment Variables section below to see the default locations.

## Examples

After installation (see below) you can do things like in the examples below. All of these examples show the `confg` command, but they also work with the `seconfg` and `syconfg` variants.

1. add your .vimrc file

`confg add ~/.vimrc`

2. make a commit

`confg commit -m "this is a commit"`

3. add a remote

`confg remote add myserver myserver:/repos/config/confg`

4. stage all changed files

`confg add -u`

If you are on a different computer (with access to the primary remote repo you have been pushing to) you can get the files with (this is the main point of this whole endeavor):

5. `confg remote add <name> <url>` and then `confg pull`

## Installation

1. clone the github repo to a convenient location
`git clone https://github.com/kcwitt/confg`
2. change into the new directory
`cd confg`
3. run it with the 'install' argument
`./confg install`

This will:

1. copy the script to '/usr/local/bin/confg'
2. create a symlink from '/usr/local/bin/seconfg' to '/usr/local/bin/confg'
3. create a symlink from '/usr/local/bin/syconfg' to '/usr/local/sbin/confg'
4. create a symlink from '/usr/local/bin/aconfg' to '/usr/local/bin/confg'

The installation locations can be changed based on the following environment variables:

* **ROOT_DIR**: defaults to '/usr/local/'
* **CONFG_FILE**: defaults to '${ROOT_DIR}/bin/confg'
* **SECONFG_FILE**: defaults to '${ROOT_DIR}/bin/seconfg'
* **SYCONFG_FILE**: defaults to '${ROOT_DIR}/sbin/syconfg' (note this is the sbin dir)
* **ACONFG_FILE**: defaults to '${ROOT_DIR}/bin/aconfg'

## Environment Variables

You can customize the repo and workdir directories for each using the following environment variables (but not really sure why you would).

CONFG_REPO="${CONFG_REPO:-"${HOME}/.confg/user"}"
CONFG_DIR="${CONFG_DIR:-"${HOME}"}"

SECONFG_REPO="${SECONFG_REPO:-"${HOME}/.confg/secure"}"
SECONFG_DIR="${SECONFG_DIR:-"${HOME}"}"

SYCONFG_REPO="${SYCONFG_REPO:-"/etc/confg"}"
SYCONFG_DIR="${SYCONFG_DIR:-"/"}"

## .gitignore

In order to prevent `git status` from showing all of the untracked files you can put a '.gitignore' file in the root of each working dir with '*.*' to ignore everything except the files you explicitly add.

## post-receive hook

In my case, I want the primary remote to be on my fileserver (I view github as useful for collaboration and easy access, but never as the primary storage for a project), but in order to access my user and system config files (not the secret user config files though) from anywhere I use the following post-receive hook in the remote.

To make this work:

1. Go to the bare git repo that is your primary repo (for example on a local fileserver which is reliably backed up) and add a github project as a remote.
2. Save the script below in the 'hooks' directory in the bare repository that you are pushing to (change 'github' in the script below to whatever you want to name the remote if you decide not to name it github).

After that, anytime you do a push from one of the `confg` variants, it will automatically be pushed up to github.

``` sh
#!/usr/bin/env sh

while read -r oldrev newrev refname; do
  # convert refname from "refs/heads/next" to just "next" (for example)
  branch="$(git rev-parse --symbolic --abbrev-ref "${refname}")"

  if [ "${branch}" = 'master' ] || [ "${branch}" = 'next' ]; then
    # specify the ssh key to use (which is in the keys directory in the repo root directory)
    # make sure to add this as a "Deploy Key" in the actual github project repo
    # https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys
    export GIT_SSH_COMMAND="ssh -i '../keys/id_ed25519'"

    git push github "${branch}"
  fi
done
```

## Setting things up

In my case, I want the primary repos to be located on my local fileserver, but also want the 'confg' and 'syconfg' repos to be on github so I can access them from anywhere for convienence.


To accomplish this:

1. create create repos on github
2. 

## What about different configs for different operating systems or user profiles

No problem, just use branches, and checkout the branch than suits whatever computer you are working on. For instance one branch for fedora and another branch for ubunto.
