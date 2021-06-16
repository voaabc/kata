---
layout: post
title: What is a container?
categories: Docker
tags: Docker
---

* content
{:toc}




### Processes 进程

Containers are just normal Linux Processes with additional configuration applied. Launch the following Redis container so we can see what is happening under the covers.
> 容器只是具有额外配置的普通Linux进程。启动下面的Redis容器，这样我们就可以看到到底发生了什么。

`docker run -d --name=db redis:alpine`

The Docker container launches a process called `redis-server`. From the host, we can view all the processes running, including those started by Docker.
> Docker容器启动一个名为`redis-server`的进程。从主机上，我们可以看到所有正在运行的进程，包括那些由Docker启动的进程。


Docker can help us identify information about the process including the PID (Process ID) and PPID (Parent Process ID) via `docker top db`
> Docker可以通过`docker top db`帮助我们识别有关包括PID（进程ID）和PPID（父进程ID）的进程的信息

Who is the PPID? Use `ps aux | grep <ppid>` to find the parent process. Likely to be Containerd.
> PPID是谁？使用`ps aux | grep<ppid>`查找父进程。可能是容器。

The command pstree will list all of the sub processes. See the Docker process tree using `pstree -c -p -A $(pgrep dockerd)`
> 命令pstree将列出所有子进程。使用`pstree-c-p-A$（pgrep dockerd）`查看Docker进程树

As you can see, from the viewpoint of Linux, these are standard processes and have the same properties as other processes on our system.
> 如您所见，从Linux的角度来看，这些是标准进程，并且具有与我们系统上的其他进程相同的属性。

#### Process Directory 进程目录
Linux is just a series of magic files and contents, this makes it fun to explore and navigate to see what is happening under the covers, and in some cases, change the contents to see the results.
> Linux只是一系列神奇的文件和内容，这使得探索和浏览幕后发生了什么变得很有趣，在某些情况下，更改内容来查看结果。

The configuration for each process is defined within the /proc directory. If you know the process ID, then you can identify the configuration directory.
> 每个进程的配置在/ proc目录中定义。 如果您知道进程ID，则可以识别配置目录。

The command below will list all the contents of `/proc`, and store the Redis PID for future use.
> 下面的命令将列出/ proc的所有内容，并存储redis pid以供将来使用。

`DBPID=$(pgrep redis-server)`
`echo Redis is $DBPID`
`ls /proc`

Each process has it's own configuration and security settings defined within different files. 
> 每个进程都具有在不同文件中定义的自己的配置和安全设置。

`ls /proc/$DBPID`

For example, you can view and update the environment variables defined to that process. 
> 例如，您可以查看和更新为该进程定义的环境变量。

`cat /proc/$DBPID/environ`
`docker exec -it db env`

### Namespaces 命名空间
One of the fundamental parts of a container is namespaces. The concept of namespaces is to limit what processes can see and access certain parts of the system, such as other network interfaces or processes.
> 容器的一个基本部分是命名空间。 命名空间的概念是限制可以看到哪些进程和访问系统的某些部分，例如其他网络接口或进程。

When a container is started, the container runtime, such as Docker, will create new namespaces to sandbox the process. By running a process in it's own Pid namespace, it will look like it's the only process on the system.
> 当容器启动时，容器运行时（如Docker）将创建新的命名空间来对进程进行沙盒处理。通过在自己的Pid命名空间中运行一个进程，它看起来就像是系统中唯一的进程。

The available namespaces are:
> 可用的命名空间包括：

Mount (mnt)

Process ID (pid)

Network (net)

Interprocess Communication (ipc)

UTS (hostnames)

User ID (user)

Control group (cgroup)

More information at https://en.wikipedia.org/wiki/Linux_namespaces

#### Unshare can launch "contained" processes.
Without using a runtime such as Docker, a process can still operate within it's own namespace. One tool to help is unshare.
> 如果不使用Docker之类的运行时，进程仍然可以在自己的命名空间中运行。一个有用的工具是unshare。

`unshare --help`

With unshare it's possible to launch a process and have it create a new namespace, such as Pid. By unsharing the Pid namespace from the host, it looks like the bash prompt is the only process running on the machine. 
> 使用unshare可以启动一个进程并让它创建一个新的名称空间，比如Pid。通过从主机取消共享Pid名称空间，看起来bash提示符是机器上运行的唯一进程。

`sudo unshare --fork --pid --mount-proc bash`
`ps`
`exit`

#### What happens when we share a namespace?
Under the covers, Namespaces are inode locations on disk. This allows for processes to shared/reused the same namespace, allowing them to view and interact.
> 实际上，命名空间是磁盘上的inode位置。这允许进程共享/重用同一个命名空间，允许它们查看和交互。

List all the namespaces with `ls -lha /proc/$DBPID/ns/`
> 列出所有名称空间 `ls -lha /proc/$DBPID/ns/`

Another tool, NSEnter is used to attach processes to existing Namespaces. Useful for debugging purposes.
> 另一个工具，NSEnter用于将进程附加到现有命名空间。 用于调试目的。

`nsenter --help`

`nsenter --target $DBPID --mount --uts --ipc --net --pid ps aux`

With Docker, these namespaces can be shared using the syntax container:<container-name>. For example, the command below will connect nginx to the DB namespace.
> 在Docker中，可以使用语法container:<container-name>共享这些命名空间。例如，下面的命令将nginx连接到DB命名空间。

```
docker run -d --name=web --net=container:db nginx:alpine
WEBPID=$(pgrep nginx | tail -n1)
echo nginx is $WEBPID
cat /proc/$WEBPID/cgroup
```

While the net has been shared, it will still be listed as a namespace. 
> 虽然网络已被共享，但它仍将作为命名空间列出。

`ls -lha /proc/$WEBPID/ns/`

However, the net namespace for both processes points to the same location.
> 但是，两个进程的net命名空间指向相同的位置

`ls -lha /proc/$WEBPID/ns/ | grep net`
`ls -lha /proc/$DBPID/ns/ | grep net`

### Chroot
An important part of a container process is the ability to have different files that are independent of the host. This is how we can have different Docker Images based on different operating systems running on our system.
> 容器进程的一个重要部分是能够拥有独立于主机的不同文件。这就是我们如何在系统上运行基于不同操作系统的不同Docker映像。

Chroot provides the ability for a process to start with a different root directory to the parent OS. This allows different files to appear in the root.
> Chroot提供了一种能力，让进程可以从父操作系统的不同根目录开始。这允许不同的文件出现在根目录中。

By writing to the file, we can change the limit limits of a process.
> 通过写入文件，我们可以更改进程的限制范围。

`echo 8000000 > /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes`

If you read the file back, you'll notice it's been converted to 7999488. 
> 如果您读回该文件，您会注意到它已被转换为7999488。

`cat /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes`

When checking Docker Stats again, the memory limit of the process is now 7.629M 
> 再次检查Docker Stats时，该进程的内存现在限制为7.629M

`docker stats db --no-stream`

### Seccomp / AppArmor
All actions with Linux is done via syscalls. The kernel has 330 system calls that perform operations such as read files, close handles and check access rights. All applications use a combination of these system calls to perform the required operations.
> Linux的所有操作都是通过系统调用完成的。内核有330个系统调用来执行诸如读取文件、关闭句柄和检查访问权限等操作。所有应用程序都使用这些系统调用的组合来执行所需的操作。

AppArmor is a application defined profile that describes which parts of the system a process can access.
> AppArmor是应用程序定义的概要文件，描述进程可以访问系统的哪些部分。

It's possible to view the current AppArmor profile assigned to a process via `cat /proc/$DBPID/attr/current`
> 可以通过`cat/proc/$DBPID/attr/current`查看分配给进程的当前AppArmor配置文件

The default AppArmor profile for Docker is docker-default (enforce).
> Docker默认的AppArmor配置文件是Docker -default (enforce)。

Prior to Docker 1.13, it stored the AppArmor Profile in /etc/apparmor.d/docker-default (which was overwritten when Docker started, so users couldn't modify it). After v1.13, Docker now generates docker-default in tmpfs, uses apparmor_parser to load it into kernel, then deletes the file
> 在Docker1.13之前，它将AppArmor配置文件存储在/etc/AppArmor.d/Docker-default中（Docker启动时会被覆盖，因此用户无法修改它）。在v1.13之后，Docker现在在tmpfs中生成Docker默认值，使用Apparmoru解析器将其加载到内核中，然后删除该文件

The template can be found at 
> 模板可在以下位置找到：

https://github.com/moby/moby/blob/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/profiles/apparmor/template.go

Seccomp provides the ability to limit which system calls can be made, blocking aspects such as installing Kernel Modules or changing the file permissions.
> Seccomp提供了限制可以进行哪些系统调用的功能，阻止安装内核模块或更改文件权限等方面。

The default allowed calls with Docker can be found at
> 默认允许的Docker调用可以在

https://github.com/moby/moby/blob/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/profiles/seccomp/default.json

When assigned to a process it means the process will be limited to a subset of the ability system calls. If it attempts to call a blocked system call is will recieve the error "Operation Not Allowed".
> 当分配给一个进程时，这意味着该进程将被限制到能力系统调用的一个子集。如果它试图调用一个阻塞的系统调用，则会收到错误“操作不允许”。

The status of SecComp is also defined within a file.
> SecComp的状态也在文件中定义。

`cat /proc/$DBPID/status`

`cat /proc/$DBPID/status | grep Seccomp`

The flag meaning are: 0: disabled 1: strict 2: filtering
> 标志含义为：0:禁用 1:严格 2:过滤


#### Capabilities 功能
Capabilities are groupings about what a process or user has permission to do. These Capabilities might cover multiple system calls or actions, such as changing the system time or hostname.
> 功能是关于进程或用户有权做什么的分组。这些功能可能包含多个系统调用或操作，例如更改系统时间或主机名。

The status file also containers the Capabilities flag. A process can drop as many Capabilities as possible to ensure it's secure.
> 状态文件还包含Capabilities标志。一个进程可以丢弃尽可能多的功能以确保它的安全性。

`cat /proc/$DBPID/status | grep ^Cap`

The flags are stored as a bitmask that can be decoded with capsh
> 这些标志存储为位掩码，可以用capsh进行解码

`capsh --decode=00000000a80425fb`


```sh
$ docker run -d --name=db redis:alpine
Unable to find image 'redis:alpine' locally
alpine: Pulling from library/redis
540db60ca938: Pull complete 
29712d301e8c: Pull complete 
8173c12df40f: Pull complete 
8cc52074f78e: Pull complete 
aa7854465cce: Pull complete 
6ab1d05b4973: Pull complete 
Digest: sha256:eaaa58f8757d6f04b2e34ace57a71d79f8468053c198f5758fd2068ac235f303
Status: Downloaded newer image for redis:alpine
2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
$ ps aux | grep redis-server
999       1085  1.3  1.0  29156 11164 ?        Ssl  03:14   0:00 redis-server *:6379
root      1127  0.0  0.0  14220   932 pts/0    S+   03:14   0:00 grep --color=auto redis-server
$ docker top db
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
999                 1085                1069                1                   03:14               ?                   00:00:00            redis-server *:6379
$ ps aux | grep 1069
root      1069  0.0  0.4   7516  4424 ?        Sl   03:14   0:00 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc -debug
root      1150  0.0  0.0  14220   968 pts/0    S+   03:14   0:00 grep --color=auto 1069
$ pstree -c -p -A $(pgrep dockerd)
dockerd(683)-+-docker-containe(725)-+-docker-containe(1069)-+-redis-server(1085)-+-{bio_aof_fsync}(1123)
             |                      |                       |                    |-{bio_close_file}(1122)
             |                      |                       |                    |-{bio_lazy_free}(1124)
             |                      |                       |                    `-{jemalloc_bg_thd}(1125)
             |                      |                       |-{docker-containe}(1071)
             |                      |                       |-{docker-containe}(1072)
             |                      |                       |-{docker-containe}(1073)
             |                      |                       |-{docker-containe}(1074)
             |                      |                       |-{docker-containe}(1075)
             |                      |                       `-{docker-containe}(1076)
             |                      |-{docker-containe}(727)
             |                      |-{docker-containe}(728)
             |                      |-{docker-containe}(729)
             |                      |-{docker-containe}(730)
             |                      |-{docker-containe}(731)
             |                      |-{docker-containe}(733)
             |                      |-{docker-containe}(738)
             |                      `-{docker-containe}(1070)
             |-{dockerd}(695)
             |-{dockerd}(696)
             |-{dockerd}(697)
             |-{dockerd}(717)
             |-{dockerd}(726)
             |-{dockerd}(735)
             |-{dockerd}(739)
             |-{dockerd}(761)
             |-{dockerd}(1027)
             `-{dockerd}(1103)
$ DBPID=$(pgrep redis-server)
$ echo Redis is $DBPID
Redis is 1085
$ ls /proc
1     13   173  228  32   5    61   72   9          execdomains  kpagecgroup   partitions     timer_list
10    130  174  23   33   515  62   721  acpi       fb           kpagecount    sched_debug    timer_stats
1069  131  18   24   34   52   63   725  buddyinfo  filesystems  kpageflags    schedstat      tty
1085  133  19   242  35   53   64   732  bus        fs           loadavg       scsi           uptime
11    134  2    25   36   54   65   736  cgroups    interrupts   locks         self           version
1155  135  20   26   4    55   66   779  cmdline    iomem        mdstat        slabinfo       version_signature
12    136  202  27   433  56   67   781  consoles   ioports      meminfo       softirqs       vmallocinfo
124   14   21   270  436  57   68   8    cpuinfo    irq          misc          stat           vmstat
125   141  211  28   438  58   683  85   crypto     kallsyms     modules       swaps          zoneinfo
126   15   218  29   440  59   686  86   devices    kcore        mounts        sys
127   16   22   3    454  6    7    87   diskstats  keys         mtrr          sysrq-trigger
128   17   223  30   458  60   704  882  dma        key-users    net           sysvipc
129   171  226  31   473  607  715  895  driver     kmsg         pagetypeinfo  thread-self
$ ls /proc/$DBPID
attr        comm             fd        map_files   net            pagemap      sessionid  status
autogroup   coredump_filter  fdinfo    maps        ns             personality  setgroups  syscall
auxv        cpuset           gid_map   mem         numa_maps      projid_map   smaps      task
cgroup      cwd              io        mountinfo   oom_adj        root         stack      timers
clear_refs  environ          limits    mounts      oom_score      sched        stat       uid_map
cmdline     exe              loginuid  mountstats  oom_score_adj  schedstat    statm      wchan
$ pgrep redis-server
1085
$ cat /proc/$DBPID/environ
*:6379
$ docker exec -it db env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=2bbbfcd5a8b6
TERM=xterm
REDIS_VERSION=6.2.4
REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-6.2.4.tar.gz
REDIS_DOWNLOAD_SHA=ba32c406a10fc2c09426e2be2787d74ff204eb3a2e496d87cff76a476b6ae16e
HOME=/root
$ unshare --help

Usage:
 unshare [options] <program> [<argument>...]

Run a program with some namespaces unshared from the parent.

Options:
 -m, --mount[=<file>]      unshare mounts namespace
 -u, --uts[=<file>]        unshare UTS namespace (hostname etc)
 -i, --ipc[=<file>]        unshare System V IPC namespace
 -n, --net[=<file>]        unshare network namespace
 -p, --pid[=<file>]        unshare pid namespace
 -U, --user[=<file>]       unshare user namespace
 -f, --fork                fork before launching <program>
     --mount-proc[=<dir>]  mount proc filesystem first (implies --mount)
 -r, --map-root-user       map current user to root (implies --user)
     --propagation slave|shared|private|unchanged
                           modify mount propagation in mount namespace
 -s, --setgroups allow|deny  control the setgroups syscall in user namespaces

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see unshare(1).
$ sudo unshare --fork --pid --mount-proc bash
$ ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
    9 pts/0    00:00:00 ps
$ exit
exit
$ ls -lha /proc/$DBPID/ns/
total 0
dr-x--x--x 2 999 packer 0 Jun 16 03:14 .
dr-xr-xr-x 9 999 packer 0 Jun 16 03:14 ..
lrwxrwxrwx 1 999 packer 0 Jun 16 03:19 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 ipc -> ipc:[4026532157]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 mnt -> mnt:[4026532155]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 net -> net:[4026532160]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 pid -> pid:[4026532158]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 user -> user:[4026531837]
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 uts -> uts:[4026532156]
$ nsenter --help

Usage:
 nsenter [options] <program> [<argument>...]

Run a program with namespaces of other processes.

Options:
 -t, --target <pid>     target process to get namespaces from
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -p, --pid[=<file>]     enter pid namespace
 -U, --user[=<file>]    enter user namespace
 -S, --setuid <uid>     set uid in entered namespace
 -G, --setgid <gid>     set gid in entered namespace
     --preserve-credentials do not touch uids or gids
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -F, --no-fork          do not fork before execing <program>
 -Z, --follow-context   set SELinux context according to --target PID

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see nsenter(1).
$ nsenter --target $DBPID --mount --uts --ipc --net --pid ps aux
PID   USER     TIME  COMMAND
    1 redis     0:00 redis-server *:6379
   16 root      0:00 ps aux
$ docker run -d --name=web --net=container:db nginx:alpine
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
540db60ca938: Already exists 
0ae30075c5da: Pull complete 
9da81141e74e: Pull complete 
b2e41dd2ded0: Pull complete 
7f40e809fb2d: Pull complete 
758848c48411: Pull complete 
Digest: sha256:ee666987e353081b53c4fc44ae1e6484ced6303cd70909bb6f1ceeca5a3f9ab4
Status: Downloaded newer image for nginx:alpine
ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
$ WEBPID=$(pgrep nginx | tail -n1)
$ echo nginx is $WEBPID
nginx is
$ cat /proc/$WEBPID/cgroup
cat: /proc//cgroup: No such file or directory
$ docker run -d --name=web --net=container:db nginx:alpine
docker: Error response from daemon: Conflict. The container name "/web" is already in use by container "ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2". You have to remove (or rename) that container to be able to reuse that name.
See 'docker run --help'.
$ WEBPID=$(pgrep nginx | tail -n1)
$ echo nginx is $WEBPID
nginx is 1328
$ cat /proc/$WEBPID/cgroup
11:perf_event:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
10:blkio:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
9:hugetlb:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
8:devices:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
7:pids:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
6:freezer:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
5:cpuset:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
4:cpu,cpuacct:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
3:net_cls,net_prio:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
2:memory:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
1:name=systemd:/docker/ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2
$ ls -lha /proc/$WEBPID/ns/
total 0
dr-x--x--x 2 systemd-network systemd-journal 0 Jun 16 03:20 .
dr-xr-xr-x 9 systemd-network systemd-journal 0 Jun 16 03:19 ..
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 ipc -> ipc:[4026532225]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 mnt -> mnt:[4026532223]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 net -> net:[4026532160]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 pid -> pid:[4026532226]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 user -> user:[4026531837]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 uts -> uts:[4026532224]
$ ls -lha /proc/$WEBPID/ns/ | grep net
dr-x--x--x 2 systemd-network systemd-journal 0 Jun 16 03:20 .
dr-xr-xr-x 9 systemd-network systemd-journal 0 Jun 16 03:19 ..
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 ipc -> ipc:[4026532225]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 mnt -> mnt:[4026532223]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 net -> net:[4026532160]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 pid -> pid:[4026532226]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 user -> user:[4026531837]
lrwxrwxrwx 1 systemd-network systemd-journal 0 Jun 16 03:20 uts -> uts:[4026532224]
$ ls -lha /proc/$DBPID/ns/ | grep net
lrwxrwxrwx 1 999 packer 0 Jun 16 03:14 net -> net:[4026532160]
$ cat /proc/$DBPID/cgroup
11:perf_event:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
10:blkio:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
9:hugetlb:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
8:devices:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
7:pids:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
6:freezer:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
5:cpuset:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
4:cpu,cpuacct:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
3:net_cls,net_prio:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
2:memory:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
1:name=systemd:/docker/2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c
$ ls /sys/fs/cgroup/
blkio  cpuacct      cpuset   freezer  memory   net_cls,net_prio  perf_event  systemd
cpu    cpu,cpuacct  devices  hugetlb  net_cls  net_prio          pids
$ cat /sys/fs/cgroup/cpu,cpuacct/docker/$DBID/cpuacct.stat
user 44
system 48
$ cat /sys/fs/cgroup/cpu,cpuacct/docker/$DBID/cpu.shares
1024
$ ls /sys/fs/cgroup/memory/docker/
2bbbfcd5a8b6d593f99973ee047b9361f86d015a3323dd5c8fb311a53c92781c  memory.kmem.usage_in_bytes
cgroup.clone_children                                             memory.limit_in_bytes
cgroup.event_control                                              memory.max_usage_in_bytes
cgroup.procs                                                      memory.move_charge_at_immigrate
ec153704e058c3be06746b60700865120e8be5aa993cb123f1bce81e69a645d2  memory.numa_stat
memory.failcnt                                                    memory.oom_control
memory.force_empty                                                memory.pressure_level
memory.kmem.failcnt                                               memory.soft_limit_in_bytes
memory.kmem.limit_in_bytes                                        memory.stat
memory.kmem.max_usage_in_bytes                                    memory.swappiness
memory.kmem.slabinfo                                              memory.usage_in_bytes
memory.kmem.tcp.failcnt                                           memory.use_hierarchy
memory.kmem.tcp.limit_in_bytes                                    notify_on_release
memory.kmem.tcp.max_usage_in_bytes                                tasks
memory.kmem.tcp.usage_in_bytes
$ DBID=$(docker ps --no-trunc | grep 'db' | awk '{print $1}')
$ WEBID=$(docker ps --no-trunc | grep 'nginx' | awk '{print $1}')
$ ls /sys/fs/cgroup/memory/docker/$DBID
cgroup.clone_children           memory.kmem.tcp.failcnt             memory.oom_control
cgroup.event_control            memory.kmem.tcp.limit_in_bytes      memory.pressure_level
cgroup.procs                    memory.kmem.tcp.max_usage_in_bytes  memory.soft_limit_in_bytes
memory.failcnt                  memory.kmem.tcp.usage_in_bytes      memory.stat
memory.force_empty              memory.kmem.usage_in_bytes          memory.swappiness
memory.kmem.failcnt             memory.limit_in_bytes               memory.usage_in_bytes
memory.kmem.limit_in_bytes      memory.max_usage_in_bytes           memory.use_hierarchy
memory.kmem.max_usage_in_bytes  memory.move_charge_at_immigrate     notify_on_release
memory.kmem.slabinfo            memory.numa_stat                    tasks
$ docker stats db --no-stream
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
2bbbfcd5a8b6        db                  0.17%               6.758MiB / 992.1MiB   0.68%               1.3kB / 0B          0B / 0B             5
$ echo 8000000 > /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes
$ cat /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes
7999488
$ docker stats db --no-stream
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
2bbbfcd5a8b6        db                  0.18%               6.758MiB / 7.629MiB   88.58%              1.3kB / 0B          0B / 0B             5
$ cat /proc/$DBPID/attr/current
docker-default (enforce)
$ cat /proc/$DBPID/status
Name:   redis-server
State:  S (sleeping)
Tgid:   1085
Ngid:   0
Pid:    1085
PPid:   1069
TracerPid:      0
Uid:    999     999     999     999
Gid:    1000    1000    1000    1000
FDSize: 64
Groups: 1000 1000 
NStgid: 1085    1
NSpid:  1085    1
NSpgid: 1085    1
NSsid:  1085    1
VmPeak:    29156 kB
VmSize:    29156 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:     11164 kB
VmRSS:     11164 kB
VmData:    23204 kB
VmStk:       132 kB
VmExe:      1672 kB
VmLib:      1656 kB
VmPTE:        56 kB
VmPMD:        12 kB
VmSwap:        0 kB
HugetlbPages:          0 kB
Threads:        5
SigQ:   0/3824
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000000001001
SigCgt: 00000000000044ea
CapInh: 00000000a80425fb
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
Seccomp:        2
Cpus_allowed:   3
Cpus_allowed_list:      0-1
Mems_allowed:   00000000,00000001
Mems_allowed_list:      0
voluntary_ctxt_switches:        4411
nonvoluntary_ctxt_switches:     180
$ cat /proc/$DBPID/status | grep Seccomp
Seccomp:        2
$ cat /proc/$DBPID/status | grep ^Cap
CapInh: 00000000a80425fb
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
$ capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
$ 
```