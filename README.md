## Synopsis

repo-snapshots is a small collection of bash scripts for taking snapshots of directories.  

The scripts were written specifically to work with apt-mirror to create and serve snapshots for local mirrors of Ubuntu APT repositories.  However, they could be fairly easy to adapt for other types of repositories.

## Motivation

In order to provide consistent patch sets to servers, a somewhat popular technique is to create snapshots of a local mirror.  No projects existed that provided tools to do simple snapshots, or document how to do it manually.

A number of similar projects do exist to provide similar functionality, or achieve a similar objective.  Some are listed at the bottom, in case you're looking for something that fits your needs better.

## Overview

## Full Working Example

These are the complete steps to setup a server to create and host snapshots. It uses apt-mirror, Apache, and repo-snapshots. 

In this example, a local mirror of the Ubuntu trusty and trusty-updates repositories will be created.  For simplicity, the mirror will be limited to the **main** component.  In addition, only the **i386** and **amd64** architectures will be mirrored.

### Install apt-mirror, apache, and repo-snapshots

Install the software.  For this project, simply checkout the git repository.
```
# apt-get install apt-mirror apache2 git
# git clone https://github.com/Chatham/repo-snapshots.git /usr/local/src/repo-snapshots
```

### Configure apt-mirror

One essential point is that apt-mirror must be configured with its **unlink** option.  This is an undocumented feature.  We also set the **_autoclean** option.  Since the snapshots have all the files, there is no reason not to clean.

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

There is no need to configure Apache or repo-snapshots by default.  For this example, we will leave them as stock setup.

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

## Going Beyond the Basics

### Rolling Patch Process

### Handling Problem Patches

### Handling Out-of-Band Patches

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
