---
layout: post
title:  "Excluding Folders Within Dropbox from Syncing"
tags: dropbox ubuntu linux bash shell
comments: true
---

{% include license.html %}

While I was setting up a laptop with [Ubuntu 19.10](http://releases.ubuntu.com/19.10/) for development, Dropbox was consuming 100% of a CPU core and devouring battery each time I created a new Python virtual environment with [virtualenv](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/). While not the end of the world, it was annoying and seemed like an easy enough problem to solve. The issue was that Dropbox was trying to sync a bunch of generated directories.

Dropbox offers a feature called [Selective Sync](https://help.dropbox.com/installs-integrations/sync-uploads/selective-sync-overview), but this requires more manual intervention than I was hoping for. There were also a [few](https://superuser.com/questions/853236/how-to-make-dropbox-ignore-specific-folders-in-the-sync) _  [similar](https://www.dropboxforum.com/t5/Dropbox/Ignore-folder-without-selective-sync/idi-p/5926) _  [questions](https://medium.com/@sherwino/how-do-you-ignore-specific-folders-like-node-modules-recursively-on-dropbox-c74ba9f2abce) and a [handful](https://arshaw.com/exclude-node-modules-dropbox-google-drive) _ 
[of](https://gist.github.com/idleberg/6c8a563e248103baaa20) _  [solutions](https://peter.grman.at/ignore-node_modules-in-dropbox/), but none I felt very comfortable using, so I decided to create my own.

## Requirements

- Must not lose data
- Should not require manual intervention, except during setup
- Should require minimal configuration
- Should minimize unnecessary synchronization by Dropbox
- Should be simple for others to audit and use

I thought that this would be a good opportunity to do some shell scripting. Since I already have `.gitignore` files in each of my Git repositories, along with a [global gitignore](https://help.github.com/en/github/using-git/ignoring-files#create-a-global-gitignore), and Git repositories were common culprits, I decided on a solution that relied on Git ignores. After poking around a bit, I came across [this answer](https://stackoverflow.com/a/467053) which identifies files that Git is currently ignoring via `git status --ignored -v`.

## Open Questions

There are two questions we need to answer:
1. How will we prevent Dropbox from syncing a particular file/directory?
2. How will we detect new files/directories that are created?

### Preventing Dropbox from Syncing
- Now that Dropbox [no longer follows symlinks](https://help.dropbox.com/installs-integrations/sync-uploads/symlinks) that go outside of Dropbox-synced directories, we could move ignored directories out and create symlinks from Dropbox to those locations
    + Pros: cross-platform (at least OSX and Linux)
    + Cons: could interfere with programs that don't follow symlinks
- Use `dropbox exclude` from the CLI
    + Pros: purpose-built solution provided by Dropbox
    + Cons: won't work on OSX or Windows
- Only sync data *into* Dropbox that we want backed up (e.g. using rsync)
    + Pros: cross-platform
    + Cons: duplicates files and possibility of data loss

### Detecting new Directories
- `inotifywatch`/`fswatch`
    + Pros: picks up new directories immediately
    + Cons: can quickly hit limits on number of inotify watches, could be a battery drain if not careful, non-trivial setup
- `cron`
    + Pros: picks up new directories fairly quickly (~60s)
    + Cons: may result in 60s of unnecessary Dropbox syncing
- Manually
    + Pros: simplest approach
    + Cons: requires manually running each time a new repository or virtual environment is created
    
## Decisions

I ended up choosing to go the `dropbox exclude` and `cron` route. You can find the [source code for the solution on Github](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/sync_gitignores_dropbox.sh).

While I ended up with a slightly more complex script than I was hoping for, there is a fairly small and easily auditable subset of code which needs to be correct to ensure that data is not lost. Further, it uses a defense-in-depth approach to allow for auto-recovery when things go wrong and a manual recovery for when things go wrong-er. One other piece to call out is that I opted to use environment variables rather than [getopts](https://www.tldp.org/LDP/abs/html/internal.html), but if I did it again I'd probably choose getopts.

## Set Up

I have tested the script on Ubuntu 19.10/Bash 5.0.3/Dropbox 84.4.170 (Dropbox CLI 2019.02.14)/Git 2.20.1. As much as possible, I tried to make it portable, but since the Dropbox CLI is limited to Linux, it won't work in its current form on OSX, and is likely to run into some issues on other Linux distros since it hasn't been tested there.

### Download the script

#### Copy and Paste
The simplest option for getting the script is to just open [it on Github](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/sync_gitignores_dropbox.sh), copy, and paste it into a local file.

#### "Curl"ing the Script
The second simplest option, provided you have `curl` installed, is to run something like

```sh
curl https://raw.githubusercontent.com/joshmcgrath08/dropbox_scripts/master/sync_gitignores_dropbox.sh > "$HOME/sync_gitignores_dropbox.sh"
```

#### Checking Out the Git Repo
The last option is to clone the [Git repository](https://github.com/joshmcgrath08/dropbox_scripts). Github has great instructions [here](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/cloning-a-repository).

### Run Manually
I recommend that you run it manually the first time for two reasons. First, it is always a good idea to do a dry-run first. Second, the first time it runs (not as a dry-run) will take significantly longer than subsequent times because of the volume of slow network calls to Dropbox. These are intentionally serial in order to minimize the potential for data loss.

Assuming `$HOME/Dropbox` is the directory Dropbox is syncing and `$HOME/sync_gitignores_dropbox.sh` is the location of the script, the following will perform an interactive dry-run:

```sh
env DRY_RUN=true DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/sync_gitignores_dropbox.sh"
```

When you type "y", you will see the exact commands to be run, but they will not actually run. You can see other options in the [README](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/README.md) or the top of the [script source](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/sync_gitignores_dropbox.sh).

Once you are satisfied with the results, you can drop the `DRY_RUN=true` and run for real.

### Setting Up Cron
```sh
crontab -e
# Select the editor of your choice

# Update $HOME/Dropbox and $HOME/sync_gitignores_dropbox.sh
# to point to the root of your Dropbox directory and script respectively

# Add the following line to the end of the file and save:
*/5 * * * * env FORCE=true DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/sync_gitignores_dropbox.sh"
```

This will run the command (and skip all interactive bits) once every five minutes.

If you want to run it once an hour instead, you could use
```sh
0 * * * * env FORCE=true DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/sync_gitignores_dropbox.sh"
```

{% include comments.html %}

