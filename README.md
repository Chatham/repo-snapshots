## Synopsis

**repo-snapshots** is a small collection of bash scripts for taking snapshots of directories.  

The scripts were written specifically to work with apt-mirror to create and serve snapshots for local mirrors of Ubuntu APT repositories.  However, they could be fairly easy to adapt for other types of repositories.

## Motivation

In order to provide consistent patch sets to servers, a somewhat popular technique is to create snapshots of a local mirror.  No projects existed that provided tools to do simple snapshots, or document how to do it manually.

The goals were to simply
- provide a consistent patch set to servers
- allow multiple patch sets to be presented, so lower environments can be used to test patches before deploying to higher environments

A number of similar projects do exist to provide similar functionality, or achieve a similar objective.  Some are listed at the bottom, in case you're looking for something that fits your needs better.

## Overview

Whenever the local mirror is updated, a snapshot copy is created using `cp -al`.  This means the snapshot directory will only create hard links to the same files as the local mirror.

As the local mirror updates over time, it will unlink the removed package versions.  The snapshot directory will retain the link, and it will be possible to acess the repository as it existed at any point when a snapshot was taken.

Since APT repositories are just a directory structure, it is easy to server the local mirrors and snapshots.  For extra simplicity, URLs can be configured to be aliases to specific snapshot directories.

## Quick Start Guide

These are the complete steps to setup a server to create and host snapshots. It uses apt-mirror, Apache, and repo-snapshots. 

In this example, a local mirror of the Ubuntu trusty and trusty-updates repositories will be created.  For simplicity, the mirror will be limited to the **main** component.  In addition, only the **i386** and **amd64** architectures will be mirrored.

It is recommended to perform the installation on Ubuntu Trusty as well.

### Install apt-mirror, apache, and repo-snapshots

Install the software.  For this project, simply checkout the git repository.
```
# apt-get install apt-mirror apache2 git
# git clone https://github.com/Chatham/repo-snapshots.git /usr/local/src/repo-snapshots
```

### Configure apt-mirror

One essential point is that apt-mirror must be configured with its **unlink** option.  The **_autoclean** option is also used.  Since the snapshots have all the files, there is no reason not to clean.

The postmirror script will be used to run the snapshot script.

`$EDITOR /etc/apt/mirror.list`
```
############# config ##################
#
# set base_path    /var/spool/apt-mirror
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
set _autoclean 1
set unlink 1
#
############# end config ##############

deb-i386 http://archive.ubuntu.com/ubuntu trusty main
deb-i386 http://archive.ubuntu.com/ubuntu trusty-updates main
deb-amd64 http://archive.ubuntu.com/ubuntu trusty main
deb-amd64 http://archive.ubuntu.com/ubuntu trusty-updates main

clean http://archive.ubuntu.com/ubuntu
```

`$EDITOR ~apt-mirror/var/postmirror.sh`
```
#!/usr/bin/env bash

/usr/local/src/repo-snapshots/snapshot.sh
```

There is no need to configure Apache or repo-snapshots by default.  For this example, leave them as stock setup.

### Perform an intial sync

Warning: This will take a long time to run, and will use alot of disk space.
```
# /usr/local/src/repo-snapshots/apt-mirror-helper.sh
```

### Configure cron job

Create a cron job to update the local repository.  A wrapper script is used so the Apache configuration can be done as root.

`$EDITOR /etc/cron.d/apt-mirror`
```
# update local mirror every 4 hours.  Ubuntu will update the official repos 6 times a day
01 */4 * * * root /usr/local/src/repo-snapshots/apt-mirror-helper.sh > /dev/null
```

### Browse the Local Mirror
At this point, the local mirror will be browseable
- http://127.0.0.1/mirror

Also, all created snapshots will be browseable.  There is no link to this location in the parent directory since the URL is an alias.
- http://127.0.0.1/mirror/snapshots

APT can be ocnfigured to point to the local mirror, or to a specific snapshot.

For example, to use the local mirror
`$EDITOR /etc/apt/sources.list`
```
deb http://127.0.0.1/mirror/archive.ubuntu.com_ubuntu trusty main
deb http://127.0.0.1/mirror/archive.ubuntu.com_ubuntu trusty-updates main
```

These URLs aren't very friendly to work with, so the next step could be to configure repo-snapshot to create URL aliases to the repo directories you want to use.

## Going Beyond the Basics

### Rolling Patch Process

The primary objective of the repo-snapshots setup is to allow consistent patch sets to be deployed.  These patch sets are represented by a repository snapshot.  Since many snapshots are available, different patch sets can be deployed to different sets of servers.

For this example, 3 patch sets will be created using familiar APT terms
- unstable will point to the latest snapshot
- testing will point to a given day
- stable will point to the previous testing snapshot

The testing snapshot will be used for **dev** servers.  The **stable** snapshot will be used for **prod** servers.

For simplicity, the servers will be patched on the first of each month, and a snapshot will always exist on that day.

#### Configure repo-snapshots

URL mappings must be defined for our named snapshots.  The dates are made up for this example.

`$EDITOR /usr/local/src/repo-snapshots/snapshot.conf`
```
declare -a URL=(
unstable archive.ubuntu.com_ubuntu/
testing archive.ubuntu.com_ubuntu/2015/03/01
stable archive.ubuntu.com_ubuntu/2015/02/01
)
```

#### Perform patching

With the URL aliases created for the named snapshots, the appropriate servers need to be pointed at the correct URL.  It is suggested that configuration management be used to automate this.

- example **dev** sources.list
```
deb http://localrepo/mirror/testing trusty main
deb http://localrepo/mirror/testing trusty-updates main
```

- example **prod** sources.list
```
deb http://localrepo/mirror/stable trusty main
deb http://localrepo/mirror/stable trusty-updates main
```

When patching is done during this month
- all dev servers will receive the exact same patches
- all prod servers will receive the exact same patches
- the prod servers will receive the patches dev installed the prevoius month

#### Rolling forward

For the next patch window, simply progress the named snapshots forward.  The prod snapshot will always be the previous dev snapshot.

`$EDITOR /usr/local/src/repo-snapshots/snapshot.conf`
```
declare -a URL=(
unstable archive.ubuntu.com_ubuntu/
testing archive.ubuntu.com_ubuntu/2015/04/01
stable archive.ubuntu.com_ubuntu/2015/03/01
)
```

### Handling Problem Patches

Patching from Ubuntu's official repositories is usually very safe, but problems do occur.  If there were never any problems, all this work to create patch sets would be unnecessary work.

When a bad package is discovered on a computer, the suggested process is
- Update the named snapshot to point to an older dated snapshot where the package is ok
- Force a downgrade
- Pin or Hold the bad package so it will not automatically update
..- This could vary with circumstances.  Pinning is done via apt_preferences, while a hold can be done with dpkg
- Resume normal patching procedures

When the package is fixed, the configuration to keep it from updating can be removed.

It is important to keep your system up to date.  A bad package is not a good reason to abandon the patching process!

### Handling Out-of-Band Patches

A somewhat common issue is that an urgent security patch will come out in the middle of your patch period.

In order to simplify this, a nice trick is to use the **unstable** named snapshot.  This can be added to all servers, and be given a lower priority to prevent automatic installation of packages.

In order to lower the **unstable** priority, it is suggested that a different DNS name be used.  The DNS name will point to the same host as the standard name, but allows us to configure apt_preferences.  For example, if your local mirror server is named **localrepo**, create a CNAME **localrepounstable** that points to **localrepo**.

`$EDITOR /etc/apt/sources.list`
```
deb http://localrepounstable/mirror/unstable trusty main
deb http://localrepounstable/mirror/unstable trusty-updates main
deb 
```

`$EDITOR /etc/apt/preferences.d/20unstable`
```
Package: *
Pin: origin localrepounstable
Pin-Priority: 400
```

With the unstable repository in place as a lower priority, all the latest packages are available for installation but they won't be installed automatically.  The process for these security patches becomes simply
- force an update by specifying the exact version of the package to install

Depending on circumstances, there may be times where it is easier to
- remove the apt_preferences priority configuration
- install the latest version of the affected packages
- restore the apt_preferences priority configuration

## Contributors

Let people know how they can dive into the project, include important links to things like issue trackers, irc, twitter accounts if applicable.

## Similar Projects
- aptly
- pakrat
- landscape
- satellite / spacewalk
- Debian snapshots

## License

A short snippet describing the license (MIT, Apache, etc.)
