---
layout: post
title: Running of Folding@home clients when computer is idle with xscreensaver
comments: true
---

## Preamble (if you know what F@H is skip this part)

I'm a big fan of [Folding@home](http://folding.stanford.edu/) (F@H) project that aims to model process of protein folding. Since the structure of proteins is extremely complex that modelling requires enormous amount of CPU power. But instead of using giant supercomputer the calculations are distributed among the community of volunteers.

To be a volunteer one needs to install a small Folding@home client that receives tasks from central server over the internet, performs a task with the lowest CPU priority (so it doesn't harm user experience) and sends results back to the server. The whole process is simple from a volunteer's perspective and doesn't require any intervention. Clients for Windows and Linux are available.

Besides serving the Greater Good there's certain sport interest because the rating of users is built according to number of units they've solved.

## Problem

Although the client runs with lowest CPU priority some tasks use considerable amount of RAM. Moreover I run 2 clients simultaneously to utilize both cores of my Core 2 Duo processor. And sometimes lack of memory caused by the clients harms user experience.

A perfect solution for that problem gives Vista's scheduler that allows to run a task when a user is not present at the computer (detecting keyboard and mouse events and CPU usage). For some time I couldn't find similar solution for Linux but later I was [suggested](http://stackoverflow.com/questions/622367/scheduling-in-linux-run-a-task-when-computer-is-idle-no-user-input/622420#622420) to hook up screensaver events and run the task when Linux screensaver is running. At first I decided to write my own screensaver but reading man pages for [xscreensaver](http://www.jwz.org/xscreensaver/) I've found that there's a possibility to detect starting and stopping of screensaver and perform arbitrary actions in such cases. The rest of this article describes this approach.

## Solution

This section contains scripts that are necessary to solve the problem. So first of all I would like to present their directory structure and briefly describe each of them:

```sh
/home/wheleph/bin/fah
  |-core1 # installation directory of the first client
    |-fah6
    |...
  |-core2 # installation directory of the second client
    |-fah6
    |...
  |-start_fah.sh # script that starts both clients
  |-stop_fah.sh # script that stops both clients
  |-control_fah.pl # listens to xscreensaver and starts/stops clients
```

So now let's proceed...

### 1. Install xscreensaver

Get and install [xscreensaver](http://www.jwz.org/xscreensaver/) if you don't have it yet and make sure that it's autostarted.

### 2. Write shell scripts that start and stop F@H client instance(s)

This script starts my two clients:

```sh
cd /home/wheleph/bin/fah/core1/
./fah6&

cd /home/wheleph/bin/fah/core2/
./fah6&
```

This scripts stops them:

```sh
PROC=$(ps axu | grep fah6 | grep -v grep | awk '{ print $2 }')
if [[ ! -z $PROC ]] ; then
  kill $PROC
fi
```

### 3. Write script that listens to xscreensaver and starts/stops F@H clients

`xscreensaver-command -watch` prints to standard output "BLANK" when screensaver is started and "UNBLANK" when it's stopped. So it's possible to write a program that reads output of this command and starts and stops F@H clients using the scripts given above.

Here's Perl script that implements such approach:

```perl
#!/usr/bin/perl

my $blanked = 0;
open (IN, "xscreensaver-command -watch |");
while (<in>) {
  if (m/^(BLANK|LOCK)/) {
    if (!$blanked) {
      # stop in case it was run manually
      system "bash -c /home/wheleph/bin/fah/stop_fah.sh > /dev/null";

      system "bash -c /home/wheleph/bin/fah/start_fah.sh > /dev/null";
      $blanked = 1;
    }
  } elsif (m/^UNBLANK/) {
    system "bash -c /home/wheleph/bin/fah/stop_fah.sh > /dev/null";
    $blanked = 0;
  }
}
```

You should include this script in a list of startup applications.

That it. You can use this approach to run any program in such manner.
