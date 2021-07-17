+++ author = "Andrei Pöhlmann"
title = "Why can't I execute this file as root on Linux?"
date = "2021-07-17"
description = ""
tags = [
"internals",
"tlpi"
]
categories = [
"linux"
]
+++

# Is the file executable?

If you are familiar with Linux, then you have probably heard that a root-privileged process 
is granted all access when it's checked against permissions.

Now, **all**-statements are in general very powerful and you want to make sure that it's really 
true before you state them (or at least people in maths and physics expect you that). And so it turns out the above statement is not entirely true.
 There is one specific, pathological case: on Linux, execute permission is
granted to a root-privileged process only if at least one of the other permission categories &mdash;
i.e. **user**, **group** or **other** &mdash; has the executable permission granted.

I've been working with Linux for a few years now and I don't remember if I've ever encountered this problem. I
certainly did when trying to execute a script with a regular **user** (most people probably forgot
at one point or another to `chmod +x script.sh`), but I wasn't aware that **root** also requires it.

So when I read about it in _The Linux Programming Interface_ (Ch. 15.4.2 Permission on Directories > 
Permission checking for privileged process) I was somehow surprised that I didn't know about it, so
I had to try it out myself.

# Example 

 Here's a simple bash script (note, the file is currently not executable):
```shell script
apoehlmann@nano:~/workspace/misc/tlpi/ch14$ ls -l 
total 4
-rw-rw-r-- 1 apoehlmann apoehlmann 24 Jul 18 00:22 hello.sh
```
with
```shell script
#!/bin/bash

echo hello
```
results in
```bash
apoehlmann@nano:~/workspace/misc/tlpi/ch14$ sudo su
root@nano:/home/apoehlmann/workspace/misc/tlpi/ch14# ./hello.sh
bash: ./hello.sh: Permission denied
```

So you really have to grant executable permission to one category, e.g. **group**:

```shell script
root@nano:/home/apoehlmann/workspace/misc/tlpi/ch14# chmod 614 hello.sh 
root@nano:/home/apoehlmann/workspace/misc/tlpi/ch14# ls -l
total 4
-rw---xr-- 1 apoehlmann apoehlmann 24 Jul 18 00:22 hello.sh
root@nano:/home/apoehlmann/workspace/misc/tlpi/ch14# ./hello.sh 
hello
```

# Implementation specific

**Note**: This is Linux specific! On some UNIX implementations root can still execute a file even when
the executable permission is not granted to any category. I was curious and tried it on MacOS, however 
it shows the same behaviour as on Linux.

#### Resources

Kerrisk, Michael. The Linux programming interface: a Linux and UNIX system programming handbook. No Starch Press, 2010.