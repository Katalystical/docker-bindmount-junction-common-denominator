# Docker for Windows Bind-mounts with junctions/inotify Bug Reproduction

## Short summary

There are cases of bind-mounts on Docker for Windows, where inode-related changes are not propagated.
To be more precise, it seems to be related to similar paths having a **common denominator which exists as a directory itself** and which are junctioned on the NTFS filesystem inside or outside the project root.

## Affected platforms

Windows 10 Pro, Docker for Windows (at least V4, several versions).
Latest confirmed version: **4.1.\*** up to **4.3.1 (72247)**
Docker Engine: **20.10.11**

**_Not_** tested using WSL Backend!

Tested on a system using **2.1.0.5** which works flawlessly, but is missing inotify events. Changes **do** propagate correctly with that version.

## How to

If you cloned the repo successfully, open up a PowerShell or Windows Terminal using a PS
and go to the repo directory.

1. `cd adummy-test`
2. Create the needed NTFS junctions by executing `.\create_junctions.bat`
3. Run the environment via `docker-compose up`
4. Go through each folder in adummy-targetdirs and do something (edit a file, create one, etc.)
5. If finished **or testing different approaches** you might need to `docker-compose down` to make sure Docker internally releases paths that are watched.
6. Remove junctions `.\remove_junctions.bat`

## What to expect

You should notice that for every first directory/new 'common denominator' all events and changes to files/directories propagate correctly.
Here an overview of what is expected, if we edit the existing file.txt in every respective subfolder.

Directory|Change propagation|Note
---------|------|----
a-dummy|:heavy_check_mark:|
a-dummy2|:x:|Common denominator `a-dummy`
a-dummy3|:x:|Common denominator `a-dummy`
adummy|:heavy_check_mark:|
adummy2|:x:|Common denominator `adummy`
aZZZdummy|:heavy_check_mark:|
aZZZdummy2|:x:|Common denominator `aZZZdummy`
xdummy|:heavy_check_mark:|
xdummyA|:x:|Common denominator `xdummy`
z|:heavy_check_mark:|
zdummyA|:x:|Common denominator `z`
zdummyB|:x:|Common denominator `z`

Your console output should look a little bit like this (depending on the mechanism/command/editor used for changing files) if you go through the directories in alphabetical order:

```
inotify-monitor_1  | Setting up watches.  Beware: since -r was given, this may take a while!
inotify-monitor_1  | Watches established.
inotify-monitor_1  | /mnt/self/junctions/a-dummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/a-dummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/adummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/adummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/aZZZdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/aZZZdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/xdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/xdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/xdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/xdummy/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/z/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/z/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/z/ MODIFY file.txt
inotify-monitor_1  | /mnt/self/junctions/z/ MODIFY file.txt
```

We're missing several changes here.

## A wild guess

This seems like a string comparison gone wrong, related to watched-paths checks inside Docker Desktop. This could also be an insufficient substr-comparison while iterating over those 'watched' junctioned directories, a dependency issue or something completely off.
While changes propagate correctly, if the data/directories/files are **fully** 'self-contained' and you're either using junction targets that are **inside** the project root or no junctions at all, this issue comes into effect if the junction target itself (or a parent directory) is **not bind-mounted** explicitly.

## Additional notes

You might also watch the `com.docker.backend.exe.log` in `%APPDATA%\Docker\log\host` which seems to list some interesting data and some erroneous looking entries related to `startVolumeWatching` and `UnlockDirectories`/`LockDirectories`.
These Docker-internal commands/code components are undocumented and might be closed source, it seems this not debuggable as a non-contributor.

## Contents

- `adummy-targetdirs` (dir) contains to-be-junctioned directories
- `adummy-test` (dir) is a docker-compose project root
- `adummy-test\Dockerfile` is a custom Dockerfile providing inotifywait which recursively watches fs events inside the container
- `adummy-selfcontained` is a variant (same data and containers, slightly different bind mounts) w/o junctioning directories outside the project root (regular)

A secondary container/service is available in the `docker-compose.yml` files, because `inotifywait` won't give you any results/events if your Docker version doesn't provide it. Therefore, I've provided a container that simply 'watches' using `tail -F` for every file.txt in every junctioned dir.
