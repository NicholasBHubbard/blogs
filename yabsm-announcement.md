I am the author of [Yabsm (yet another btrfs snapshot manager)](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm), which is [written in Perl](https://metacpan.org/dist/App-Yabsm/source/bin/yabsm), and I will explain how I use Yabsm to manage my Btrfs snapshots.

This article is meant to supplement the official documentation linked above, and assumes a basic understanding of Linux's [Btrfs](https://en.wikipedia.org/wiki/Btrfs) filesystem.

Please note that Yabsm can be configured to suit many different use cases other than the one described here.


<a id="orgfe91a36"></a>

# Snapshots vs Backups

Before we go on, lets clear up the difference between a snapshot and a backup.

A snapshot is a read-only nested [subvolume](https://btrfs.readthedocs.io/en/latest/Subvolumes.html) created with a command such as `btrfs subvolume snapshot -r $SUBVOLUME $DEST`. **SNAPSHOTS ARE NOT RELIABLE BACKUPS!** If a subvolume is corrupted then all snapshots of that subvolume will also be corrupted.

A backup is an [incremental backup](https://btrfs.wiki.kernel.org/index.php/Incremental_Backup) sent to some location via Btrfs's [send/receive](https://btrfs.readthedocs.io/en/latest/Send-receive.html) commands. These backups will not be corrupted if the subvolume being backed up is corrupted.


<a id="org449a345"></a>

# My Btrfs Filesystem

I like to have just one top-level Btrfs subvolume mounted at `/`. This allows me to snapshot my entire system (excluding nested subvolumes) by running `btrfs subvolume snapshot -r / $DEST`.


<a id="org028bea9"></a>

# My Yabsm Config

My configuration is based on the philosophy that because snapshots are both valuable and cheap, it makes sense to take a lot of snapshots.

Here is my `/etc/yabsm.conf`:

    yabsm_dir=/.snapshots/yabsm
    
    subvol root_subvol {
        mountpoint=/
    }
    
    snap root {
        subvol=root_subvol
    
        timeframes=5minute,hourly,daily
        5minute_keep=36
        hourly_keep=72
        daily_times=15:00,23:59
        daily_keep=62
    }
    
    ssh_backup slackmac {
        subvol=root_subvol
        ssh_dest=slackmac
        dir=/.snapshots/yabsm-slacktop
    
        timeframes=daily
        daily_times=23:59
        daily_keep=365
    }
    
    local_backup easystore {
        subvol=root_subvol
        dir=/mnt/easystore/backups/yabsm-slacktop
    
        timeframes=daily
        daily_times=23:59
        daily_keep=365
    }


<a id="orga0d51da"></a>

### Yabsm Dir

I use the traditional `/.snapshots/yabsm` directory as my `yabsm_dir`, which is the location that my snapshots will reside. Yabsm will also use this directory for storing data necessary for performing SSH and local backups.


<a id="org40ab1c1"></a>

### Subvol

As I mentioned earlier, I only have one top-level Btrfs subvolume, so I only need to define one `subvol` in my Yabsm config, which I name `root_subvol`.


<a id="org77c9f84"></a>

### Snap

I define one [snap](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm#Snaps) named *root* that tells Yabsm I want to take snapshots of `root_subvol` in the *5minute*, *hourly*, and *daily* [timeframe categories](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm#Timeframes).

In the *5minute* timeframe category I keep 36 snapshots. This lets me go to any state of my machine in the last 3 hours in 5 minute increments. I use the *5minute* category because it gives me a valuable safety net. How many times have you broken your code that was working 20 minutes ago? If you take *5minute* snapshots then you can easily go back to the state of that code 20 minutes ago.

In the *hourly* timeframe I keep 72 snapshots, which allows me to go back 3 days in hourly increments. How many times have you broken code that was working 2 days ago? If you take *hourly* snapshots (and keep enough of them), you can go back through the state of your machine from 2 days ago, in 1 hour increments.

In the *daily* timeframe category I keep 62 snapshots taken at midnight (23:59) and midafternoon (15:00). This gives me two snapshots per day in the last month.

Please note that there is also a `weekly` and `monthly` timeframe category.


<a id="org41e4eb4"></a>

### SSH Backup

I define one [ssh_backup](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm#SSH-Backups) named *slackmac* that backs up my system to my old MacBook running Slackware.

The *ssh_dest* value is set to *slackmac*, which is a host defined in the *yabsm* user's `$HOME/.ssh/config` file. (Yabsm runs as a daemon process, using the special username `yabsm`.)

The *dir* value is set to the directory on *slackmac* where the backups will be located.

I perform this *ssh_backup* only in the *daily* timeframe category, backing up every night at midnight. I keep 365 of these backups so I can go back an entire year.


<a id="orgc5909cb"></a>

### Local Backup

I define one [local_backup](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm#Local-Backups) named `easystore` that backs up my system to my EasyStore external hard drive.

The hard drive is mounted at `/mnt/easystore`, and I keep my backups in the `/backups/yabsm-slacktop` directory on the hard drive.

Just like my `slackmac` *ssh_backup*, I perform my *local_backup* only in the `daily` timeframe category, every night at midnight.


<a id="orgb9f813e"></a>

# Finding Snapshots

Yabsm provides the [find](https://metacpan.org/dist/App-Yabsm/view/bin/yabsm#Finding-Snapshots) command that I use to jump around to different snapshots and backups. The *find* command takes two arguments, the first is the name of any of your *snaps*, *ssh_backups*, or *local_backups*. The second argument is a query. The different kinds of queries are all documented in the link above.

Instead of repeating the documentation, let's break down a practical example of the *find* command's usage.

How many times have you broken code that worked 30 minutes ago? Because I take *5minute* snapshots I can easily get back the state of the code 30 minutes ago.

An example:

    $ diff $HOME/projects/foo/foo.sh "$(yabsm find root back-30-mins)/$HOME/projects/foo/script.sh"

This command will show the `diff` output of the `$HOME/projects/foo/foo.sh` file with this same file that was snapshotted 30 minutes ago. We can use this output to help figure out what we messed up.

The command `yabsm find root back-30-mins` will output the path to a snapshot for the *snap* named *root* that was taken 30 minutes ago. In the example we use our shell's [parameter expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html) feature to create a string that appends the path to `foo.sh` to the output of the `yabsm find` command. This is a powerful pattern!

The find command can do more than find a snapshot taken N units ago, it can also:

-   Find the newest or oldest snapshot/backup.
-   Find a snapshot/backup taken on a specific day and time.
-   Find all the snapshots/backups taken before or after a certain time.
-   Find all the snapshots/backups taken between two times.
-   Find all snapshots/backups.

The output of `yabsm find --help` shows some examples:

    usage: yabsm <find|f> [--help] [<SNAP|SSH_BACKUP|LOCAL_BACKUP> <QUERY>]
    
    see the section "Finding Snapshots" in 'man yabsm' for a detailed explanation on
    how to find snapshots and backups.
    
    examples:
        yabsm find home_snap back-10-hours
        yabsm f root_ssh_backup newest
        yabsm f home_local_backup oldest
        yabsm f home_snap 'between b-10-mins 15:45'
        yabsm f root_snap 'after back-2-days'
        yabsm f root_local_backup 'before b-14-d'


<a id="orgdac6334"></a>

# Synopsis

Yabsm is a powerful tool for managing your Btrfs snapshots. If you are interested in using Yabsm, then I recommend you consult the [official documentation](https://metacpan.org/release/NHUBBARD/App-Yabsm-3.12/view/bin/yabsm).

