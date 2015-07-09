---
layout: post
title: On Linux manpages
comments: true
---

Today I've upgraded Scala on my Ubuntu 13.04 laptop and just for the sake of curiosity looked inside its folder structure. Among other things I've found a directory with man pages:

```sh
$ ls
bin  doc  examples  lib  man  misc  src

$ ls man/man1/
fsc.1  scala.1  scalac.1  scaladoc.1  scalap.1
```

Then I ran `man scala` and surprisingly saw the man page from the distribution. Well... that's cool but how could man utility possibly know about completely custom location? I've just unzipped the archive, defined `SCALA_HOME` and added `$SCALA_HOME/bin` into my `PATH`.

After some googling I've found the explanation. `manpath` utility displays the path that is actually used by `man`:

```sh
$ manpath
/home/wheleph1304/bin/scala-2.10.2/man:/usr/local/man:/usr/local/share/man:/usr/share/man
```

All right, I see that for some reason it picked the correct directory. To find out more details we can provide additional debug key:

```sh
$ manpath -d
# LOTS OF OUTPUT
path directory /home/wheleph1304/bin/scala-2.10.2/bin is not in the config file
but does have a ../man, man, ../share/man, or share/man subdirectory
adding /home/wheleph1304/bin/scala-2.10.2/man to manpath
# LOTS OF OUTPUT
```

And now it's all clear. `man` examines the `PATH` entries and looks for man pages around them in some well-known directories.

That's it. Not a big deal, but it was interesting to go off on a tangent into such small investigation.
