---
layout: post
title:  "Excluding Folders Within Dropbox from Syncing"
tags: dropbox ubuntu linux bash shell
comments: true
---

While I was setting up a laptop with [Ubuntu 19.10](http://releases.ubuntu.com/19.10/) for development, Dropbox was consuming 100% of a CPU core and devouring battery each time I set up a new Python virtual environment with [virtualenv](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/). While not the end of the world, it did seem like a waste to let Dropbox synchronize  directories that I could recreate with a single command. The same issue exists for `node_modules` in npm projects and build artifacts in just about any coding project.

Dropbox offers a feature called [Selective Sync](https://help.dropbox.com/installs-integrations/sync-uploads/selective-sync-overview), but this would require futzing around in the Dropbox UI each time I created a new environment. There exist a [few](https://superuser.com/questions/853236/how-to-make-dropbox-ignore-specific-folders-in-the-sync) _  [similar](https://www.dropboxforum.com/t5/Dropbox/Ignore-folder-without-selective-sync/idi-p/5926) _  [questions](https://medium.com/@sherwino/how-do-you-ignore-specific-folders-like-node-modules-recursively-on-dropbox-c74ba9f2abce) and a [handful](https://arshaw.com/exclude-node-modules-dropbox-google-drive) _ 
[of](https://gist.github.com/idleberg/6c8a563e248103baaa20) _  [solutions](https://peter.grman.at/ignore-node_modules-in-dropbox/).

At the end of the day, I wasn't comfortable with the idea of mastering my data elsewhere and periodically syncing it into Dropbox, as this seemed antithetical to Dropbox's spirit and risked losing data. I also knew that `dropbox exclude` tends to delete the files/directories to be ignored, which I didn't see properly handled in the script-based solutions. Further, since I already use Git heavily, I wanted a solution that could reuse my existing `.gitignore` files.

## Requirements

- **Must not** lose data
- **Should not** require manual intervention, except during setup
- **Should** reuse existing `.gitignore` configuration
- **Should** minimize resource usage, including unnecessary synchronization by Dropbox
- **Should** be simple for others to audit and use
- **Could** be cross-platform (i.e. OSX and Windows in addition to Linux)

## Open Questions

There are two questions I still need to answer:
1. How to prevent Dropbox from syncing a particular file/directory?
2. How to detect new files/directories that are created?

### Preventing Dropbox from Syncing
- Now that Dropbox [no longer follows symlinks](https://help.dropbox.com/installs-integrations/sync-uploads/symlinks) that go outside of Dropbox-synced directories, I could move ignored directories out and create symlinks from Dropbox to those locations
    + Pros: cross-platform (at least OSX and Linux)
    + Cons: could interfere with programs that don't follow symlinks
- Use `dropbox exclude` from the CLI
    + Pros: purpose-built solution provided by Dropbox
    + Cons: only works on Linux
- Only sync data *into* Dropbox that I want backed up (e.g. using rsync)
    + Pros: cross-platform
    + Cons: duplicates files and possibility of data loss

### Detecting new Directories
- `inotifywatch` or `fswatch`
    + Pros: picks up new directories immediately
    + Cons: can quickly hit limits on number of inotify watches, could be a battery drain if not careful, non-trivial setup
- `cron`
    + Pros: picks up new directories fairly quickly (~60s)
    + Cons: some amount of unnecessary syncing
    
## Decisions

I will quickly rule out syncing data into a Dropbox directory, as I'm not comfortable with the composition of syncing processes, particularly when it comes to the possibility of data loss. I want to explore symlinks to see if it offers a workable approach for OS X. For now, though, using `dropbox exclude` is the simplest approach that works on Linux.

`inotifywatch` and `fswatch` are interesting, as they use the same approach that Dropbox uses to know when files change. Unfortunately, neither ship with Ubuntu. While it's not a big deal to install them via `apt-get`, I would prefer it work out of the box on all Linux distros. `cron`, however, is simple to use and comes bundled with most Linux distros.

You can find my `cron`/`dropbox exclude` solution [on Github](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/sync_gitignores_dropbox.sh) in the form of a shell script.

While I ended up with a slightly more complex script than I was hoping for, there is a fairly small and easily auditable subset of code which needs to be correct to ensure that data is not lost. Further, it uses a defense-in-depth approach to allow for auto-recovery when things go wrong and a manual recovery for when things go wrong-er. One other piece to call out is that I opted to use environment variables rather than [getopts](https://www.tldp.org/LDP/abs/html/internal.html), but if I did it again I'd probably choose getopts.

## Setup
I have tested the script on Ubuntu 19.10/Bash 5.0.3/Dropbox 84.4.170 (Dropbox CLI 2019.02.14)/Git 2.20.1. Hopefully it works without too much fuss on other Linux distros, but the current use of `dropbox exclude` means it won't work on OS X.

### Download the script

Clone the [Git repository](https://github.com/joshmcgrath08/dropbox_scripts). Github has great instructions [here](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/cloning-a-repository) if you're unfamiliar with how.

If you don't have Git installed or don't want to clone the repository, you can just download the script [here](https://raw.githubusercontent.com/joshmcgrath08/dropbox_scripts/master/sync_gitignores_dropbox.sh). If you do this, make sure to update the paths below accordingly.

### Run Manually
I recommend that you run it manually the first time for two reasons. First, it is always a good idea to do a dry-run first when there are hard-to-reverse effects. Second, the first time it runs (not as a dry-run) will take significantly longer because it has more than incremental work.

Assuming `$HOME/Dropbox` is the directory Dropbox is syncing and `$HOME/dropbox_scripts/sync_gitignores_dropbox.sh` is the location of the script, the following will perform an interactive dry-run:

```sh
env DRY_RUN=true DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/dropbox_scripts/sync_gitignores_dropbox.sh"
```

When you type "y", you will see the exact commands to be run, but they will not actually run. You can see other options in the [README](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/README.md) or the top of the [script source](https://github.com/joshmcgrath08/dropbox_scripts/blob/master/sync_gitignores_dropbox.sh).

Once you are satisfied with the results, you can drop the `DRY_RUN=true` and run for real:

```sh
env DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/dropbox_scripts/sync_gitignores_dropbox.sh"
```

### Run Periodically
Once you are confident running it manually, you can set it up to run periodically via `cron`.

There are several ways to set up `cron` jobs, but here I'll show how to use `crontab`.

1. Type `crontab -e` at the command line
2. If prompted, select the editor you are most comfortable with
3. Assuming `$HOME/Dropbox` is the directory Dropbox is syncing and `$HOME/dropbox_scripts/sync_gitignores_dropbox.sh` is the location of the script, add the following line to the end of the crontab, save, and exit: 
```sh
*/5 * * * * env FORCE=true DROPBOX_ROOT="$HOME/Dropbox" bash "$HOME/dropbox_scripts/sync_gitignores_dropbox.sh"
```

This will run the command (and skip all interactive bits) once every five minutes.

{% include license.html %}
{% include comments.html %}

