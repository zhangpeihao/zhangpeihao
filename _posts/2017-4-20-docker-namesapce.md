
## 深入解析Docker背后的namespace技术


> 摘要：Docker基于mamespace、cgroups、chroot等技术来构建容器，不是一个系统调用就能搞定，容器是一个用户态的概念。本文中Docker软件工程师Michael Crosby深入探讨了Docker对namespace技术的应用。
相信你在很多地方都看到过“Docker基于mamespace、cgroups、chroot等技术来构建容器”的说法，但你有没有想过为何容器的构建需要这些技术？ 为什么不是一个简单的系统调用就可以搞定？原因在于Linux内核中并不存在“linux container”这个概念，容器是一个用户态的概念。


### Namespaces

在第一部分，我会在文章中讨论Docker在使用Linux namespace时，如何创建Linux namespace。在之后的博客中我们会讨论namespace如何与其它特性如cgroups和隔离的文件系统相结合，去实现更多有用的功能。

从根本上说，namespace是Linux系统的底层概念，有一些不同类型的命名空间被部署在核内。跟踪docker run -it --privileged --net host crosbymichael/make-containers这段代码，我们就可以深入到每个不同的namespace。开始会有一些预加载文件和配置。尽管我们也会在用Docker为我们运行的容器中创建namespace，不要让他影响到你，我选择提供一个容器预加载所有依赖项的方法。我使用 --net host标志，这样可以在容器内看到host的网络接口。也需要提供--privilged标签，以保证拥有正确的权限去通过容器创建新的namespace。

以下是Dockerfile内的内容： 

```
FROM debian:jessie

RUN apt-get update && apt-get install -y \
    gcc \
    vim \
    emacs

COPY containers/ /containers/
WORKDIR /containers
CMD ["bash"]
```

我会使用C语言来解释这个例子，因为它比Go语言更容易去解释底层的细节。 

### NET Namespace

network namespaces为你的系统网络协议栈提供了自己的视图。这个协议栈包括你的本地主机（localhost）。确认你在目录crosbymichael/make-containers下，并运行 ip a查看所有运行在你的主机上的网络接口。 

```
> ip a
root@development:/containers# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:19:ca:f2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe19:caf2/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:20:84:47 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.103/24 brd 192.168.56.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe20:8447/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
    inet 172.17.42.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::5484:7aff:fefe:9799/64 scope link
       valid_lft forever preferred_lft forever
```

这就是当前在我的主机系统中的所有网络接口。现在让我们写一段代码创建一个新的network interface。为此，我们将写一个C语言库的框架，系统调用了clone。我们将从调用clone开始，文件skeleton.c应该在demo容器的工作目录中，我们将利用这个文件作为我们例子的基础。下面是例子的代码：

``` 
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sched.h>
#include <sys/wait.h>
#include <errno.h>

#define STACKSIZE (1024*1024)
static char child_stack[STACKSIZE];

struct clone_args {
        char **argv;
};

// child_exec is the func that will be executed as the result of clone
static int child_exec(void *stuff)
{
        struct clone_args *args = (struct clone_args *)stuff;
        if (execvp(args->argv[0], args->argv) != 0) {
                fprintf(stderr, "failed to execvp argments %s\n",
                        strerror(errno));
                exit(-1);
        }
        // we should never reach here!
        exit(EXIT_FAILURE);
}

int main(int argc, char **argv)
{
        struct clone_args args;
        args.argv = &argv[1];

        int clone_flags = SIGCHLD;

        // the result of this call is that our child_exec will be run in another
        // process returning it's pid
        pid_t pid =
            clone(child_exec, child_stack + STACKSIZE, clone_flags, &args);
        if (pid < 0) {
                fprintf(stderr, "clone failed WTF!!!! %s\n", strerror(errno));
                exit(EXIT_FAILURE);
        }
        // lets wait on our child process here before we, the parent, exits
        if (waitpid(pid, NULL, 0) == -1) {
                fprintf(stderr, "failed to wait pid %d\n", pid);
                exit(EXIT_FAILURE);
        }
        exit(EXIT_SUCCESS);
}
```

这是个小的C程序，可以让你执行./a.out ip a。它把你通过命令行传入的参数，作为任何你想使用的进程的参数。不用担心具体实施太多，因为我们将要做的事情将要发生有趣的变化。它将会用任何你想要的参数执行你希望的程序。这意味着如果你想执行下面的demo之一，同时它会产生一个shell会话，这样你就可以在你的namespace中“闲逛”。你可以自己的方式探索与检查这些不同的namespace。因此让我们复制这个文件，并开始使用network namespace。 

```
> cp skeleton.c network.c
在这个文件里有一个很特殊的变量，叫做clone_flags，大部分的变化将在此处发生。namespace主要由clone标志控制。network namespace的clone标记是CLONE_NEWNET。我们需要把int clone_flags = SIGCHLD;这一行改为 int clone_flags = CLONE_NEWNET | SIGCHLD;。这样调用clone就为我们创建一个新的network namespace。在network.c中保存这一修改，然后编译运行。 
> gcc -o net network.c
> ./net ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

```

这次运行的结果与首次运行ip a相比看起来很不相同。这一次我们只看到了一个 loopback接口。这是因为我们创建的进程，只有一个它自己的network namespace视图，而不是整个host。这就是如何创建一个新的network namespace的方法。

Docker使用新的network namespace启动一个veth接口，这样你的容器将拥有它自己的桥接ip地址，通常是docker0。接下来，我们不再继续讲述如何在namespace安装接口。相关内容将在另一篇文章中讲述。

### MNT Namespace

mount namespace可以让看到系统中所有挂载点在某个范围下的目录视图。人们经常把它和在chroot中禁锢进程混淆在一起，或者是认为他们是相似的，还有人说容器使用mount namespac来把进程禁锢在它的根文件系统中，这都是不对的！

让我们再来拷贝一份skeleton.c用来做挂载相关的修改。可以快速的构建和运行， 从而看下执行mount命令后我们当前的挂载点的样子。

```
> cp skeleton.c mount.c > gcc -o mount mount.c > ./mount mount
        proc on /proc type proc (rw,nosuid,nodev,noexec,relatime) tmpfs on /dev
        type tmpfs (rw,nosuid,mode=755) shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
        mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime) devpts
        on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
        sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime) /dev/disk/by-uuid/d3aa2880-c290-4586-9da6-2f526e381f41
        on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)
        /dev/disk/by-uuid/d3aa2880-c290-4586-9da6-2f526e381f41 on /etc/hostname
        type ext4 (rw,relatime,errors=remount-ro,data=ordered) /dev/disk/by-uuid/d3aa2880-c290-4586-9da6-2f526e381f41
        on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered) devpts
        on /dev/console type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
```

上面是在我的demo容器中看到的挂载点。为了创建一个新的mount namespace，我们使用CLONE_NEWNS标志位。大家可以注意到这个标志位名称有点奇怪，为什么不是 CLONE_NEWMOUNT或者CLONE_NEWMNT呢？这是因为mount namespace是Linux中的第一个命名空间，所以这里的这里的标记位参数名字有点不符合常规，经常我们在编码实现一个新的特性或者应用的时候，我们往往不能预想到最终结果全貌。不论怎样，我们就把 CLONE_NEWNS添加到clone_flags变量中，结果就是， int clone_flags = CLONE_NEWNS | SIGCHLD;，再来编译mount.c并运行同样的命令。 

```
> cp skeleton.c mount.c
> gcc -o mount mount.c
> ./mount mount
```

这次没有任何变化，为何？因为运行在新的mount namespace中的进程，在底层系统下仍然拥有一个/proc视图。结果就是新的进程继承了底层挂载点的视图。我们有一些方法来阻止出现这样的结果，比如说，使用 pivot_root，将在后续的博客中详细介绍。

然而，有一种方法，我们可以尝试在新的mount namespace上挂载点东西。比如在/mytmp下挂载新的 tmpfs。我们在C代码中执行mount命令，并把需要新挂载的挂载点作为参数写进去。为了达到这个目标，我们需要在 child_exec函数中调用execvp之前增加代码，代码如下：

```
// child_exec is the func that will be executed as the result of clone
    static int child_exec(void *stuff) { struct clone_args *args = (struct
    clone_args *)stuff; if (mount("none", "/mytmp", "tmpfs", 0, "") != 0) {
    fprintf(stderr, "failed to mount tmpfs %s\n", strerror(errno)); exit(-1);
    } if (execvp(args->argv[0], args->argv) != 0) { fprintf(stderr, "failed
    to execvp argments %s\n", strerror(errno)); exit(-1); } // we should never
    reach here! exit(EXIT_FAILURE); }
```

在编译和执行之前，我们需要创建一个目录/mytmp，并运行以上的改变。

```
> mkdir /mytmp
> gcc -o mount mount.c
> ./mount mount
# cutting out the common output...
none on /mytmp type tmpfs (rw,relatime)
```

这里去掉了一些常见的输出。

从结果中就能看出，多了一个新的tmpfs挂载点。赞！继续在当前shell下执行mount来对比下。

注意到了为何tmpfs挂载点为何没有显示出来了么？这是因为我们创建的挂载点是在我们自己的mount namespace下，不是在父namespace下。

前面我说过mount namespace和filesystem jail是不同的，继续执行我们的./mount和 ls命令，就能给出证明了。

### UTS Namespace

UTS namespace（UNIX Timesharing System包含了运行内核的名称、版本、底层体系结构类型等信息）用于系统标识。包含了hostname 和域名domainname 。它使得一个容器拥有属于自己hostname标识，这个主机名标识独立于宿主机系统和其上的其他容器。让我们开始，拷贝 skeleton.c 然后借助他运行hostname 命令。

```
> cp skeleton.c uts.c
> gcc -o uts uts.c
> ./uts hostname
development
```


这个会显示你的系统主机名（在我的案例中应该是 development）。就像先前做的，让我们添加clone标识给UTS namespace的clone_flags变量。这个变量值应该是CLONE_NEWUTS。当你编译并运行他的时候，你会看到其输出竟然是一样的。这些UTS namespace的值是继承他的母系统。好吧，也就是在这个新的namespace中，我们能修改其主机名而不会对他的宿主系统和其宿主系统的其他容器造成影响，他们有隔离的UTS namespace。

让我们在 child_exec 函数中修改hostname。为此，我们需要添加#include <unistd.h>头文件使其能访问到sethostname函数，同时还需要添加#include <string.h>头文件使得 setthostname函数能够调用 strlen 函数。修改完的 child_exec 应该如下：

``` 
// child_exec is the func that will be executed as the result of clone
static int child_exec(void *stuff)
{
        struct clone_args *args = (struct clone_args *)stuff;
        const char * new_hostname = "myhostname";
        if (sethostname(new_hostname, strlen(new_hostname)) != 0) {
                fprintf(stderr, "failed to execvp argments %s\n",
                        strerror(errno));
                exit(-1);
        }
        if (execvp(args->argv[0], args->argv) != 0) {
                fprintf(stderr, "failed to execvp argments %s\n",
                        strerror(errno));
                exit(-1);
        }
        // we should never reach here!
        exit(EXIT_FAILURE);
}
```

请确保在你的main函数中的clone_flags的变量应该是这样的nt clone_flags = CLONE_NEWUTS | SIGCHLD;，然后编译并用同样的命令参数运行。这个时候你可以看到用它执行 hostname命令的返回值。同时为了核查这个变动不会影响你当前的shell环境，我们将执行 hostname 并确认返回了先前原始的值。 

```
> gcc -o uts uts.c
> ./uts hostname
myhostname
> hostname
development
IPC Namespace
```

### IPC namespace
IPC namespace用于隔离进程间通信，像SysV的消息队列，让我们为这个命名空间创建一个skeleton.c的副本。 

```
> cp skeleton.c ipc.c
```

我们测试IPC命名空间的方法，是通过在主机上创建一个消息队列，当我们在IPC namespace中产生一个新进程时确保我们不能看到它。让我们首先在当前的shell中创建一个消息队列，运行skeleton副本代码来查看队列。

```
> ipcmk -Q
Message queue id: 65536
> gcc -o ipc ipc.c
> ./ipc ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   message
0xfe7f09d1 65536      root       644        0            0
```

不使用新的IPC namespace，你可以看到同样的消息队列被创建。现在让我们增加CLONE_NEWIPC标签给我们的 clone_flags变量，去为我们的进程创建一个新的IPC namespace。clone_flags变量可以看作是 int clone_flags = CLONE_NEWIPC | SIGCHLD;，重新编译和再次执行同样的命令： 

```
> gcc -o ipc ipc.c
> ./ipc ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   message
```

完成！子进程现在在一个新的IPC namespace中，并拥有完全独立的视图，还能访问消息队列。 
###PID Namespace

这部分是非常有趣的。PID （Process Identification，OS里指进程识别号）namespace是划分那些一个进程可以查看并与之交互的PID的方式。当我们创建一个新的 PID namespace时，第一个进程的PID会被赋值为1。进程退出时，内核会杀死这个namespace内的其他进程。让我们来通过制作 skeleton.c副本开始我们的改变。

```
> cp skeleton.c pid.c
```
创建一个新的 PID namespace，我们需要设置clone_flags为 CLONE_NEWPID. 该变量应该看起来像int clone_flags = CLONE_NEWPID | SIGCHLD;，我们在shell中运行 ps aux，然后以相同的参数编译和运行我们的pid.c二进制文件。 

```
> ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  20332  3388 ?        Ss   21:50   0:00 bash
root       147  0.0  0.1  17492  2088 ?        R+   22:49   0:00 ps aux
> gcc -o pid pid.c
> ./pid ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  20332  3388 ?        Ss   21:50   0:00 bash
root       153  0.0  0.0   5092   728 ?        S+   22:50   0:00 ./pid ps aux
root       154  0.0  0.1  17492  2064 ?        R+   22:50   0:00 ps aux
```

在我们预期中ps aux的PID是1，或至少看不到任何从其他父进程进来的 pid 。为什么会这样？我们孵化的进程仍然会有一个来自父进程的/proc视图，也就是说 /proc挂载在主机系统上。那么如何我们解决这个问题？我们如何确保我们新的进程只可以查看在它所在的namespace内的pid呢？ 我们可以通过重新挂载/proc 开始。

因为我们会用mount来处理，我们可以借此机会使用从MNT namespace所了解到的内容，并结合PID namespace，以确保我们不会将其与自己的主机系统的 /proc搞混。

我们可以通过包含为PID namespace设置的clone flag和为MNT namespace设置的clone flag来启动。他们看起来像是 int clone_flags = CLONE_NEWPID | CLONE_NEWNS | SIGCHLD;。我们需要编辑 child_exec函数和重新挂载proc。系统调用unmount和 mount即可。因为我们正在创造一个新的MNT namespace，这不会搞乱我们的主机系统。结果应如下所示：

```
// child_exec is the func that will be executed as the result of clone
        static int child_exec(void *stuff) { struct clone_args *args = (struct
        clone_args *)stuff; if (umount("/proc", 0) != 0) { fprintf(stderr, "failed
        unmount /proc %s\n", strerror(errno)); exit(-1); } if (mount("proc", "/proc",
        "proc", 0, "") != 0) { fprintf(stderr, "failed mount /proc %s\n", strerror(errno));
        exit(-1); } if (execvp(args->argv[0], args->argv) != 0) { fprintf(stderr,
        "failed to execvp argments %s\n", strerror(errno)); exit(-1); } // we should
        never reach here! exit(EXIT_FAILURE); }
```

再一次生成并运行看看会发生什么？ 

```
> gcc -o pid pid.c
> ./pid ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   9076   784 ?        R+   23:05   0:00 ps aux
```

完美 ！我们新的 PID namespace已经在MNT namespace的帮助下正常运作！ 
USER Namespace

User namespace是最新的子用户空间，它允许你创建独立于其他namespace之外的用户。这是通过GID和UID映射实现的。

这里有一个未指定映射的实例应用程序。如果我们添加CLONE_NEWUSER到clone_flags，然后运行id或ls -la，会得到nobody的输出，因为当前用户还未被创建。

```
> cp skeleton.c user.c
# add the clone flag
> gcc -o user user.c
> ./user ls -la
total 84
drwxr-xr-x 1 nobody nogroup 4096 Nov 16 23:10 .
drwxr-xr-x 1 nobody nogroup 4096 Nov 16 22:17 ..
-rwxr-xr-x 1 nobody nogroup 8336 Nov 16 22:15 mount
-rw-r--r-- 1 nobody nogroup 1577 Nov 16 22:15 mount.c
-rwxr-xr-x 1 nobody nogroup 8064 Nov 16 21:52 net
-rw-r--r-- 1 nobody nogroup 1441 Nov 16 21:52 network.c
-rwxr-xr-x 1 nobody nogroup 8544 Nov 16 23:05 pid
-rw-r--r-- 1 nobody nogroup 1772 Nov 16 23:02 pid.c
-rw-r--r-- 1 nobody nogroup 1426 Nov 16 21:59 skeleton.c
-rwxr-xr-x 1 nobody nogroup 8056 Nov 16 23:10 user
-rw-r--r-- 1 nobody nogroup 1442 Nov 16 23:10 user.c
-rwxr-xr-x 1 nobody nogroup 8408 Nov 16 22:40 uts
-rw-r--r-- 1 nobody nogroup 1694 Nov 16 22:36 uts.c
```

这是个很简单地例子，但是细想你会发现通过user namespace，可以以root权限在容器中运行（不是主机系统中的root）。不要忘记你可以随时更改 ls -la到bash中，并通过shell深入了解namespace。

## 总结

本文中，我们回顾了mount，network，user，PID，UTS和IPC Linux namespace，并没有修改太多代码，只是增加了一些flag。复杂的工作集中在管理多个内核子系统的交互。就像我一开始提到的，namespace只是我们用来创建容器的一种工具，我希望PID的例子能使我们理解，是如何使多个namespace协同来创建容器的。

在未来的博客中，我们会详细介绍如何将容器进程禁锢在root文件系统中（又称为docker镜像），以及如何使用cgroups。最后的博文中会综合介绍容器是如何被创建。