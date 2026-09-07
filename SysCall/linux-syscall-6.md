Linux 资源限制
================================================================================

系统中的每个进程都会使用一定量的资源，例如文件、CPU 时间、内存等。

这些资源并不是无限的，因此我们需要一种管理资源的手段。有时，了解某项资源的当前限制或修改它的值非常有用。本节将介绍一些工具，它们可以帮助我们获取进程的资源限制信息，以及提高或降低这些限制。

我们将先从用户空间的角度开始，然后看看它们在 Linux 内核中是如何实现的。

管理进程资源限制主要有以下 3 个基本的[系统调用](https://en.wikipedia.org/wiki/System_call)：

  * `getrlimit`
  * `setrlimit`
  * `prlimit`

前两个系统调用允许进程读取和设置系统资源的限制。最后一个是前两个系统调用的扩展，`prlimit` 允许为指定 [PID](https://en.wikipedia.org/wiki/Process_identifier) 的进程设置和读取资源限制。这些函数的定义如下。

`getrlimit` 的定义如下：

```C
int getrlimit(int resource, struct rlimit *rlim);
```

`setrlimit` 的定义如下：

```C
int setrlimit(int resource, const struct rlimit *rlim);
```

然后 `prlimit` 的定义如下：

```C
int prlimit(pid_t pid, int resource, const struct rlimit *new_limit,
            struct rlimit *old_limit);
```

前两个函数都接受以下 2 个参数：

  * `resource` - 表示资源类型（我们稍后将介绍可用的资源类型）；
  * `rlim` - 由 `soft` 和 `hard` 限制组成。

限制有以下两种类型：

  * `soft`
  * `hard`

第一种是进程某项资源的实际限制。第二种是 `soft` 限制的上限值，只能由超级用户设置。因此，`soft` 限制永远不能超过相应的 `hard` 限制。

这两个值都存储在 `rlimit` 结构中：

```C
struct rlimit {
    rlim_t rlim_cur;
    rlim_t rlim_max;
};
```

最后一个函数看起来稍微复杂一些，它接受 `4` 个参数。除了 `resource` 参数之外，它还接受：

  * `pid` - 指定执行 `prlimit` 的进程 ID；
  * `new_limit` - 如果此参数不为 `NULL`，则提供新的限制值；
  * `old_limit` - 如果此参数不为 `NULL`，则当前的 `soft` 和 `hard` 限制将被存放在此处。

[ulimit](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html#index-ulimit) 工具正是使用了 `prlimit` 函数。我们可以借助 [strace](https://linux.die.net/man/1/strace) 工具验证这一点。

例如：

```
~$ strace ulimit -s 2>&1 | grep rl

prlimit64(0, RLIMIT_NPROC, NULL, {rlim_cur=63727, rlim_max=63727}) = 0
prlimit64(0, RLIMIT_NOFILE, NULL, {rlim_cur=1024, rlim_max=4*1024}) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
```

这里可以看到 `prlimit64`，却看不到 `prlimit`。这是因为这里显示的是底层系统调用，而不是库函数调用。

现在来看看可用资源类型的列表：

| 资源              | 描述
|-------------------|------------------------------------------------------------------------------------------|
| RLIMIT_CPU        | CPU 时间限制 （以秒为单位）                                                                |
| RLIMIT_FSIZE      | 进程可以创建的文件的最大大小                                                                |
| RLIMIT_DATA       | 进程数据段的最大大小                                                                       |
| RLIMIT_STACK      | 进程栈的最大大小（以字节为单位）                                                            |
| RLIMIT_CORE       | [core](https://man7.org/linux/man-pages/man5/core.5.html) 文件的最大大小                   |
| RLIMIT_RSS        | 可以为进程分配的 RAM 字节数                                                                |
| RLIMIT_NPROC      | 用户可以创建的最大进程数                                                                   |
| RLIMIT_NOFILE     | 进程可以打开的最大文件描述符数量                                                            |
| RLIMIT_MEMLOCK    | 通过 [mlock](https://man7.org/linux/man-pages/man2/mlock.2.html) 锁定到 RAM 的最大字节数   |
| RLIMIT_AS         | 虚拟内存的最大大小（以字节为单位）                                                          |
| RLIMIT_LOCKS      | [flock](https://linux.die.net/man/1/flock) 和与锁相关的 [fcntl](https://man7.org/linux/man-pages/man2/fcntl.2.html) 调用的最大数量 |
| RLIMIT_SIGPENDING | 调用进程的用户可以排队等待的[信号](https://man7.org/linux/man-pages/man7/signal.7.html)的最大数量 |
| RLIMIT_MSGQUEUE   | 可以为 [POSIX 消息队列](https://man7.org/linux/man-pages/man7/mq_overview.7.html) 分配的字节数 |
| RLIMIT_NICE       | 进程可以设置的最大 [nice](https://linux.die.net/man/1/nice) 值                             |
| RLIMIT_RTPRIO     | 最大实时优先级的值                                                                         |
| RLIMIT_RTTIME     | 在实时调度策略下，在不进行阻塞系统调用的情况下，进程可以调度的最大微秒数                        |

如果你查看开源项目的源代码，你会发现读取或更新资源限制是非常常见的操作。

例如：[systemd](https://github.com/systemd/systemd/blob/01a45898fce8def67d51332bccc410eb1e8710e7/src/core/main.c)

```C
/* Don't limit the coredump size */
(void) setrlimit(RLIMIT_CORE, &RLIMIT_MAKE_CONST(RLIM_INFINITY));
```

又如 [haproxy](https://github.com/haproxy/haproxy/blob/25f067ccec52f53b0248a05caceb7841a3cb99df/src/haproxy.c)：

```C
getrlimit(RLIMIT_NOFILE, &limit);
if (limit.rlim_cur < global.maxsock) {
	Warning("[%s.main()] FD limit (%d) too low for maxconn=%d/maxsock=%d. Please raise 'ulimit-n' to %d or more to avoid any trouble.\n",
		argv[0], (int)limit.rlim_cur, global.maxconn, global.maxsock, global.maxsock);
}
```

我们刚刚从用户空间的角度简单了解了资源限制的相关内容，现在让我们来看看这些系统调用在 Linux 内核中的实现。

Linux 内核中的资源限制
--------------------------------------------------------------------------------

`getrlimit` 和 `setrlimit` 系统调用的实现很相似。二者都会执行 `do_prlimit` 函数。该函数是 `prlimit` 系统调用的核心实现，负责在用户空间与指定的 `rlimit` 结构之间复制数据：

`getrlimit`：

```C
SYSCALL_DEFINE2(getrlimit, unsigned int, resource, struct rlimit __user *, rlim)
{
	struct rlimit value;
	int ret;

	ret = do_prlimit(current, resource, NULL, &value);
	if (!ret)
		ret = copy_to_user(rlim, &value, sizeof(*rlim)) ? -EFAULT : 0;

	return ret;
}
```

以及 `setrlimit`：

```C
SYSCALL_DEFINE2(setrlimit, unsigned int, resource, struct rlimit __user *, rlim)
{
	struct rlimit new_rlim;

	if (copy_from_user(&new_rlim, rlim, sizeof(*rlim)))
		return -EFAULT;
	return do_prlimit(current, resource, &new_rlim, NULL);
}
```

这些系统调用的实现定义在内核源文件 [kernel/sys.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sys.c) 中。

首先，`do_prlimit` 函数会检查给定的资源是否有效：

```C
if (resource >= RLIM_NLIMITS)
	return -EINVAL;
```

检查失败时会返回 `-EINVAL` 错误。检查成功后，如果传入的新的限制值不为 `NULL`，还会执行以下两个检查：

```C
if (new_rlim) {
	if (new_rlim->rlim_cur > new_rlim->rlim_max)
		return -EINVAL;
	if (resource == RLIMIT_NOFILE &&
			new_rlim->rlim_max > sysctl_nr_open)
		return -EPERM;
}
```

检查给定的 `soft` 限制是否超过 `hard` 限制；如果给定资源是文件描述符数量，还会检查 `hard` 限制是否不大于 `sysctl_nr_open` 的值。可以通过 [procfs](https://en.wikipedia.org/wiki/Procfs) 查看 `sysctl_nr_open` 的值：

```
~$ cat /proc/sys/fs/nr_open
1048576
```

完成所有这些检查后，我们会锁定 `tasklist`，以确保在更新给定资源的限制时，与[信号](https://man7.org/linux/man-pages/man7/signal.7.html)处理程序相关的内容不会被销毁：

```C
read_lock(&tasklist_lock);
...
...
...
read_unlock(&tasklist_lock);
```

之所以需要这样做，是因为 `prlimit` 系统调用允许我们根据给定的 pid 更新另一个任务的限制。锁定任务列表后，我们取得指定进程中对应资源限制的 `rlimit` 实例：

```C
rlim = tsk->signal->rlim + resource;
```

其中，`tsk->signal->rlim` 只是一个 `struct rlimit` 数组，用于表示各种资源的限制。如果 `new_rlim` 不为 `NULL`，我们就更新限制值。如果 `old_rlim` 不为 `NULL`，则填充它：

```C
if (old_rlim)
    *old_rlim = *rlim;
```

总结
--------------------------------------------------------------------------------

这是介绍 Linux 内核中系统调用实现的第二部分的结尾。如果你有任何问题或建议，可以在 Twitter 上联系 [0xAX](https://twitter.com/0xAX)，给我发送[邮件](mailto:anotherworldofworld@gmail.com)，或者直接创建一个 [issue](https://github.com/0xAX/linux-insides/issues/new)。

链接
--------------------------------------------------------------------------------

* [系统调用](https://en.wikipedia.org/wiki/System_call)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [ulimit](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html#index-ulimit)
* [strace](https://linux.die.net/man/1/strace)
* [POSIX 消息队列](https://man7.org/linux/man-pages/man7/mq_overview.7.html)
