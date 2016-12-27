Linux 内核开发
================================================================================

简介
--------------------------------------------------------------------------------

如你所知，我从去年开始写了一系列关于 `x86_64` 架构汇编语言程序设计的[博文](http://xinqiu.gitbooks.io/categories/assembly/)。除了大学期间写过一些 `Hello World` 这样无实用价值的程序之外，我从来没写过哪怕一行的底层代码。那些程序也是很久以前的事情了，就像我刚才说的，我几乎完全没有写过底层代码。直到不久前，我才开始对这些事情感兴趣，因为我意识到我虽然可以写出程序，但是我却不知道我的程序是怎样被组织运行的。

在写了一些汇编代码之后，我开始**大致**了解了程序在编译之后会变成什么样子。尽管如此，还是有很多其他的东西我不能够理解。例如：当 `syscall` 指令在我的汇编程序内执行时究竟发生了什么，当 `printf` 函数开始工作时又发生了什么，还有，我的程序是如何通过网络与其他计算机进行通信的。[汇编](https://en.wikipedia.org/wiki/Assembly_language#Assembler)语言并没有为这些问题带来答案，于是我决定做一番深入研究。我开始学习 Linux 内核的源代码，并且尝试着理解那些让我感兴趣的东西。然而 Linux 内核源代码也没有解答我**所有的**问题，不过我自身关于 Linux 内核及其外围流程的知识确实掌握的更好了。

在我开始学习 Linux 内核的九个半月之后，我写了这部分内容，并且发布了本书的[第一部分](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html)。到现在为止，本书共包括了四个部分，而这并不是终点。我之所以写这一系列关于 Linux 内核的文章其实更多的是为了我自己。你也知道，Linux 内核的代码量极其巨大，另外还非常容易忘记这一块或那一块内核代码做了什么，或者忘记某些东西是怎么实现的。出乎意料的是 [linux-insides](https://github.com/0xAX/linux-insides) 很快就火了，并且在九个月后积攒了 `9096` 个星星：

![github](http://s2.postimg.org/jjb3s4frt/stars.png)

看起来人们对 Linux 内核的内在机制非常的感兴趣。除此之外，在我写 `linux-insides` 的这段时间里，我收到了很多人发来的问题，这些问题大都是关于如何开始向 Linux 内核贡献代码。通常来说，人们是很有兴趣为开源项目做贡献的，Linux 内核也不例外：

![google-linux](http://s4.postimg.org/yg9z5zx0d/google_linux.png)

这么看起来大家对 Linux 内核的开发流程非常感兴趣。我认为如果这么一本关于 Linux 内核的书却不包括一部分来讲讲如何参与 Linux 内核开发的话，那就非常奇怪了。这就是我决定写这篇文章的原因。在本文中，你不会看到为什么你应该对贡献 Linux 内核感兴趣，但是如果你想参与 Linux 内核开发的话，那这部分就是为你而作。

让我们开始吧。

如何入门 Linux 内核
---------------------------------------------------------------------------------

首先，让我们看看如何获取、构建并运行 Linux 内核。你可以通过两种方式来运行你自己定制的内核：

* 在虚拟机里运行 Linux 内核；
* 在真实的硬件上运行 Linux 内核。

我会对这两种方式都展开描述。在我们开始对 Linux 内核做些什么之前，我们首先需要先获取它。根据你目的的不同，有两种方式可以做到这一点。如果你只是想更新一下你电脑上的 Linux 内核版本，那么你可以使用特定于你 [Linux 发行版](https://en.wikipedia.org/wiki/Linux_distribution)的命令。

在这种情况下，你只需要使用[软件包管理器](https://en.wikipedia.org/wiki/Package_manager)下载新版本的 Linux 内核。例如，为了将 [Ubuntu (Vivid Vervet)](http://releases.ubuntu.com/15.04/) 系统的 Linux 内核更新至 `4.1` 版本，你只需要执行以下命令：

```
$ sudo add-apt-repository ppa:kernel-ppa/ppa
$ sudo apt-get update
```

在这之后，再执行下面的命令：

```
$ apt-cache showpkg linux-headers
```

然后选择你感兴趣的 Linux 内核的版本。最后，执行下面的命令并且将 `${version}` 替换为你从上一条命令的输出中选择的版本号。

```
$ sudo apt-get install linux-headers-${version} linux-headers-${version}-generic linux-image-${version}-generic --fix-missing
```

最后重启你的系统。重启完成后，你将在 [grub](https://en.wikipedia.org/wiki/GNU_GRUB) 菜单中看到新的内核。

另一方面，如果你对 Linux 内核开发感兴趣，那么你就需要获得 Linux 内核的源代码。你可以在 [kernel.org](https://kernel.org/) 网站上找到它并且下载一个包含了 Linux 内核源代码的归档文件。实际上，Linux 内核的开发流程完全建立在 `git` [版本控制系统](https://en.wikipedia.org/wiki/Version_control)之上，所以你需要通过 `git` 来从 `kernel.org` 上获取内核源代码：

```
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

我不知道你怎么看，但是我本身是非常喜欢 `github` 的。它上面有一个 Linux 内核主线仓库的[镜像](https://github.com/torvalds/linux)，你可以通过以下命令克隆它：

```
$ git clone git@github.com:torvalds/linux.git
```

我是用我自己 [fork](https://github.com/0xAX/linux) 的仓库来进行开发的，等到我想从主线仓库拉取更新的时候，我只需要执行下方的命令即可：

```
$ git checkout master
$ git pull upstream master
```

注意这个主线仓库的远程主机名叫做 `upstream`。为了将主线 Linux 仓库添加为一个新的远程主机，你可以执行：

```
git remote add upstream git@github.com:torvalds/linux.git
```

在此之后，你将有两个远程主机：

```
~/dev/linux (master) $ git remote -v
origin	git@github.com:0xAX/linux.git (fetch)
origin	git@github.com:0xAX/linux.git (push)
upstream	https://github.com/torvalds/linux.git (fetch)
upstream	https://github.com/torvalds/linux.git (push)
```

其中一个远程主机是你的 fork 仓库 (`origin`)，另一个是主线仓库 (`upstream`)。

现在，我们已经有了一份 Linux 内核源代码的本地副本，我们需要配置并编译内核。Linux 内核的配置有很多不同的方式，最简单的方式就是直接拷贝 `/boot` 目录下已安装内核的配置文件：

```
$ sudo cp /boot/config-$(uname -r) ~/dev/linux/.config
```

如果你当前的内核被编译为支持访问 `/proc/config.gz` 文件，你也可以使用以下命令复制当前内核的配置文件：

```
$ cat /proc/config.gz | gunzip > ~/dev/linux/.config
```

如果你对发行版维护者提供的标准内核配置文件并不满意，你也可以手动配置 Linux 内核，有两种方式可以做到这一点。Linux 内核的根 [Makefile](https://github.com/torvalds/linux/blob/master/Makefile) 文件提供了一系列可配置的目标选项。例如 `menuconfig` 为内核配置提供了一个菜单界面：

![menuconfig](http://s21.postimg.org/zcz48p7yf/menucnonfig.png)

 `defconfig` 参数会为当前的架构生成默认的内核配置文件，例如 [x86_64 defconfig](https://github.com/torvalds/linux/blob/master/arch/x86/configs/x86_64_defconfig)。你可以将 `ARCH` 命令行参数传递给 `make`，以此来为给定架构创建 `defconfig` 配置文件：

```
$ make ARCH=arm64 defconfig
```

`allnoconfig`、 `allyesconfig` 以及 `allmodconfig` 参数也允许你生成新的配置文件，其效果分别为尽可能多的选项都关闭、尽可能多的选项都启用或尽可能多的选项都作为模块启用。`nconfig` 命令行参数提供了基于 `ncurses` 的菜单程序来配置 Linux 内核：

![nconfig](http://s29.postimg.org/hpghikp4n/nconfig.png)

`randconfig` 参数甚至可以随机地生成 Linux 内核配置文件。我不会讨论如何去配置 Linux 内核或启用哪个选项，因为没有必要这么做：首先，我不知道你的硬件配置；其次，如果我知道了你的硬件配置，那么剩下的问题就是搞清楚如何使用程序生成内核配置，而这些程序的使用都是非常容易的。

好了，我们现在有了 Linux 内核的源代码并且完成了配置。下一步就是编译 Linux 内核了。最简单的编译 Linux 内核的方式就是执行以下命令：

```
$ make
scripts/kconfig/conf  --silentoldconfig Kconfig
#
# configuration written to .config
#
  CHK     include/config/kernel.release
  UPD     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  ...
  ...
  ...
  OBJCOPY arch/x86/boot/vmlinux.bin
  AS      arch/x86/boot/header.o
  LD      arch/x86/boot/setup.elf
  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
  Setup is 15740 bytes (padded to 15872 bytes).
System is 4342 kB
CRC 82703414
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

为了增加内核的编译速度，你可以给 `make` 传递命令行参数 `-jN`，这里的 `N` 指定了并发执行的命令数目：

```
$ make -j8
```

如果你想为一个架构构建一个与当前内核不同的内核，那么最简单的方式就是传递下面两个参数：

* `ARCH` 命令行参数是目标架构名；
* `CROSS_COMPILER` 命令行参数是交叉编译工具的前缀；

例如，如果我们想使用默认内核配置文件为 [arm64 架构](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features)编译 Linux 内核，我们需要执行以下命令：

```
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu- defconfig
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu-
```

编译的结果就是你会看到压缩后的内核文件 - `arch/x86/boot/bzImage`。既然我们已经编译好了内核，那么就可以把它安装到我们的电脑上或者只是将它运行在模拟器里。

安装 Linux 内核
--------------------------------------------------------------------------------

就像我之前写的，我们将考察两种运行新内核的方法：第一种情况，我们可以在真实的硬件上安装并运行新版本的 Linux 内核，第二种情况就是在虚拟机上运行 Linux 内核。在前面的段落中我们看到了如何从源代码来构建 Linux 内核，并且我们现在已经得到了内核的压缩镜像：

```
...
...
...
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

在我们获得了 [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage) 之后，我们需要使用以下命令来为新的 Linux 内核安装 `headers` 和 `modules`：

```
$ sudo make headers_install
$ sudo make modules_install
```

以及内核自身：

```
$ sudo make install
```

从这时起，我们已经安装好了新版本的 Linux 内核，现在我们需要通知 `bootloader` 新内核已经安装完成。我们当然可以手动编辑 `/boot/grub2/grub.cfg` 配置文件并将新内核添加进去，但是我更推荐使用脚本来完成这件事。我现在在使用两种不同的 Linux 发行版：Fedora 和 Ubuntu，有两种方式可以用来更新 [grub](https://en.wikipedia.org/wiki/GNU_GRUB) 配置文件，我目前正在使用下面的脚本来达到这一目的：

```shell
#!/bin/bash

source "term-colors"

DISTRIBUTIVE=$(cat /etc/*-release | grep NAME | head -1 | sed -n -e 's/NAME\=//p')
echo -e "Distributive: ${Green}${DISTRIBUTIVE}${Color_Off}"

if [[ "$DISTRIBUTIVE" == "Fedora" ]] ;
then
    su -c 'grub2-mkconfig -o /boot/grub2/grub.cfg'
else
    sudo update-grub
fi

echo "${Green}Done.${Color_Off}"
```

这是新 Linux 内核安装过程中的最后一步，在这之后你可以重启你的电脑，然后在启动过程中选择新版本的内核。

第二种情况就是在虚拟机内运行新的 Linux 内核，我更倾向于使用 [qemu](https://en.wikipedia.org/wiki/QEMU)。首先我们需要为此构建初始的虚拟内存盘 - [initrd](https://en.wikipedia.org/wiki/Initrd)。`initrd` 是一个临时的根文件系统，它在初始化期间被 Linux 内核使用，而那时其他的文件系统尚未被挂载。我们可以使用以下命令构建 `initrd`：

首先我们需要下载 [busybox](https://en.wikipedia.org/wiki/BusyBox)，然后运行 `menuconfig` 命令配置它：

```shell
$ mkdir initrd
$ cd initrd
$ curl http://busybox.net/downloads/busybox-1.23.2.tar.bz2 | tar xjf -
$ cd busybox-1.23.2/
$ make menuconfig
$ make -j4
```

`busybox` 是一个可执行文件 - `/bin/busybox`，它包括了一系列类似于 [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities) 的标准工具。在 `busysbox` 菜单界面上我们需要启用 `Build BusyBox as a static binary (no shared libs)` 选项：

![busysbox menu](http://s18.postimg.org/sj92uoweh/busybox.png)

我们可以按照下方的路径找到这个菜单项：

```
Busybox Settings
--> Build Options
```

之后，我们从 `busysbox` 的配置菜单退出去，然后执行下面的命令来构建并安装它：

```
$ make -j4
$ sudo make install
```

既然 `busybox` 已经安装完了，那么我们就可以开始构建 `initrd` 了。为了完成构建过程，我们需要返回到之前的 `initrd` 目录并且运行命令：

```
$ cd ..
$ mkdir -p initramfs
$ cd initramfs
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ../busybox-1.23.2/_install/* .
```

这会把 `busybox` 复制到 `bin` 目录、`sbin` 目录以及其他相关目录内。现在我们需要创建可执行的 `init` 文件，该文件将会在系统内作为第一个进程执行。我的 `init` 文件仅仅挂载了 [procfs](https://en.wikipedia.org/wiki/Procfs) 和 [sysfs](https://en.wikipedia.org/wiki/Sysfs) 文件系统并且执行了 shell 程序：

```shell
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys

exec /bin/sh
```

最后，我们创建一个归档文件，这就是我们的 `initrd` 了：

```
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ~/dev/initrd_x86_64.gz
```

我们现在可以在虚拟机里运行内核了。就像我之前写过的，我偏向于使用 [qemu](https://en.wikipedia.org/wiki/QEMU) 来完成这些工作，下面的命令可以用来运行我们的 Linux 内核：

```
$ qemu-system-x86_64 -snapshot -m 8GB -serial stdio -kernel ~/dev/linux/arch/x86_64/boot/bzImage -initrd ~/dev/initrd_x86_64.gz -append "root=/dev/sda1 ignore_loglevel"
```

![qemu](http://s22.postimg.org/b8ttyigup/qemu.png)

从现在起，我们就可以在虚拟机内运行 Linux 内核了，这意味着我们可以开始对内核进行修改和测试了。

除了上面的手动过程之外，还可以考虑使用 [ivandaviov/minimal](https://github.com/ivandavidov/minimal) 来自动生成 `initrd`。

Linux 内核开发入门
---------------------------------------------------------------------------------

这部分的核心内容主要回答了两个问题：在你发送第一个 Linux 内核补丁之前你应该做什么 (`to do`) 和不能做什么 (`not to do`)。请千万不要把应该做的事 (`to do`) 和待办事项 (`todo`) 搞混了。我无法回答你能为 Linux 内核修复什么问题，我只是想告诉你我拿 Linux 内核源代码做实验的过程。

首先，我需要使用以下命令从 Linus 的仓库中拉取最新的更新：

```
$ git checkout master
$ git pull upstream master
```

在这之后，我的本地 Linux 内核源代码仓库已经和[主线](https://github.com/torvalds/linux)仓库同步了。现在我们可以在源代码上做些修改了。就像我之前写的，关于从哪开始修改或者可以做些什么，我并不能给你太多建议。不过，对于新手来说最好的地方就是 `staging` 源码树，也就是 [drivers/staging](https://github.com/torvalds/linux/tree/master/drivers/staging) 上的驱动集合。`staging` 源码树的主要维护者是 [Greg Kroah-Hartman](https://en.wikipedia.org/wiki/Greg_Kroah-Hartman)，该源码树正是你的琐碎补丁可以被接受的地方。让我们看一个简单的例子，该例子描述了如何生成补丁、检查补丁以及如何将补丁发送到 [Linux 内核邮件列表](https://lkml.org/)。

如果我们查看一下为 [Digi International EPCA PCI](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) 基础设备所写的驱动程序，在 295 行我们将会看到 `dgap_sindex` 函数：

```C
static char *dgap_sindex(char *string, char *group)
{
	char *ptr;

	if (!string || !group)
		return NULL;

	for (; *string; string++) {
		for (ptr = group; *ptr; ptr++) {
			if (*ptr == *string)
				return string;
		}
	}

	return NULL;
}
```

这个函数查找 `group` 和 `string` 共有的字符并返回其位置。在研究 Linux 内核源代码期间，我注意到 [lib/string.c](https://github.com/torvalds/linux/blob/master/lib/string.c#L473) 文件里实现了一个 `strpbrk` 函数，该函数和 `dgap_sinidex` 函数做了同样的事。使用现存函数的另一种自定义实现并不是一个好主意，所以我们可以从 [drivers/staging/dgap/dgap.c](https://github.com/torvalds/linux/blob/master/drivers/staging/dgap/dgap.c) 源码文件中移除 `dgap_sindex` 函数并使用 `strpbrk` 替换它。

首先，让我们基于当前主分支创建一个新的 `git` 分支，该分支与 Linux 内核主仓库同步：

```
$ git checkout -b "dgap-remove-dgap_sindex"
```

然后，我们可以将 `dgap_sindex` 函数替换为 `strpbrk`。做完这些修改之后，我们需要重新编译 Linux 内核或者只重编译 [dgap](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) 目录。不要忘了在内核配置文件中启用这个驱动，你可以在如下位置找到该驱动：

```
Device Drivers
--> Staging drivers
----> Digi EPCA PCI products
```

![dgap menu](http://s4.postimg.org/d3pozpge5/digi.png)

现在是时候提交修改了，我使用下面的命令组合来完成这件事：

```
$ git add .
$ git commit -s -v
```

最后一条命令运行后将会打开一个编辑器，该编辑器会从 `$GIT_EDITOR` 或 `$EDITOR` 环境变量中进行选择。 `-s` 命令行参数会在提交信息的末尾按照提交者名字加上一行 `Signed-off-by`。你在每一条提交信息的最后都能看到这一行，例如 - [00cc1633](https://github.com/torvalds/linux/commit/00cc1633816de8c95f337608a1ea64e228faf771)。这一行的主要目的是追踪谁做的修改。`-v` 选项按照合并格式显示 `HEAD` 提交和即将进行的最新提交之间的差异。这样做不是并必须的，但有时候却很有用。再来说下提交信息，实际上，一条提交信息由两部分组成：

第一部分放在第一行，它包括了一句对所做修改的简短描述。这一行以 `[PATCH]` 做前缀，后面跟上子系统、驱动或架构的名字，以及在 `:` 之后的简述信息。在我们这个例子中，这一行信息如下所示：

```
[PATCH] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

在简述信息之后，我们通常空一行再加上对本次提交的详尽描述。在我们的这个例子中，这些信息如下所示：

```
The <linux/string.h> provides strpbrk() function that does the same that the
dgap_sindex(). Let's use already defined function instead of writing custom.
```

在提交信息的最后是 `Sign-off-by` 这一行。注意，提交信息的每一行不能超过 `80` 个字符并且提交信息必须详细地描述你所做的修改。千万不要只写一条类似于 `Custom function removed` 这样的信息，你需要描述你做了什么以及为什么这样做。补丁的审核者必须据此知道他们正在审核什么内容，除此之外，这里的提交信息本身也非常有用。每当你不能理解一些东西的时候，我们都可以使用 [git blame](http://git-scm.com/docs/git-blame) 命令来阅读关于修改的描述。

提交修改之后，是时候生成补丁文件了。我们可以使用 `format-patch` 命令来完成：

```
$ git format-patch master
0001-staging-dgap-Use-strpbrk-instead-of-dgap_sindex.patch
```

我们把分支名字 (这里是`master`) 传递给 `format-patch` 命令，该命令会根据那些包括在 `dgap-remove-dgap_sindex` 分支但不在  `master` 分支的最新改动来生成补丁。你会发现， `format-patch` 命令生成的文件包含了最新所做的修改，该文件的名字是基于提交信息的简述来生成的。如果你想按照自定义的文件名来生成补丁，你可以使用 `--stdout` 选项：

```
$ git format-patch master --stdout > dgap-patch-1.patch
```

最后一步就是在我们生成补丁之后将之发送到 Linux 内核邮件列表。当然，你可以使用任意的邮件客户端，不过 `git` 为此提供了一个专门的命令：`git send-email`。在发送补丁之前，你需要知道发到哪里。虽然你可以直接把它发送到 `linux-kernel@vger.kernel.org` 这个邮件列表，但这很可能让你的补丁因为巨大的消息流而被忽略掉。最好的选择是将补丁发送到你的修改所属子系统的维护者那里。你可以使用 `get_maintainer.pl` 这个脚本来找到这些维护者的名字。你所需要做的就是将你代码所在的文件或目录作为参数传递给脚本。

```
$ ./scripts/get_maintainer.pl -f drivers/staging/dgap/dgap.c
Lidza Louina <lidza.louina@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Mark Hounschell <markh@compro.net> (maintainer:DIGI EPCA PCI PRODUCTS)
Daeseok Youn <daeseok.youn@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Greg Kroah-Hartman <gregkh@linuxfoundation.org> (supporter:STAGING SUBSYSTEM)
driverdev-devel@linuxdriverproject.org (open list:DIGI EPCA PCI PRODUCTS)
devel@driverdev.osuosl.org (open list:STAGING SUBSYSTEM)
linux-kernel@vger.kernel.org (open list)
```

你将会看到一组姓名和与之相关的邮件地址。现在你可以通过下面的命令发送补丁了：

```
$ git send-email --to "Lidza Louina <lidza.louina@gmail.com>" \
  --cc "Mark Hounschell <markh@compro.net>"                   \
  --cc "Daeseok Youn <daeseok.youn@gmail.com>"                \
  --cc "Greg Kroah-Hartman <gregkh@linuxfoundation.org>"      \
  --cc "driverdev-devel@linuxdriverproject.org"               \
  --cc "devel@driverdev.osuosl.org"                           \
  --cc "linux-kernel@vger.kernel.org"
```

这就是全部的过程。补丁被发出去了，现在你所需要做的就是等待 Linux 内核开发者的反馈。在你发送完补丁并且维护者接受它之后，你将在维护者的仓库中看到它 (例如前文你看到的[补丁](https://git.kernel.org/cgit/linux/kernel/git/gregkh/staging.git/commit/?h=staging-testing&id=b9f7f1d0846f15585b8af64435b6b706b25a5c0b))。一段时间后，维护者将会向 Linus 发送一个拉取请求，之后你就会在主线仓库里看到你的补丁了。

这就是全部内容。

一些建议
--------------------------------------------------------------------------------

在该部分的最后，我想给你一些建议，这些建议大都是关于在 Linux 内核的开发过程中需要做什么以及不能做什么的：

* 考虑，考虑，再考虑。在你决定发送补丁之前再三考虑。

* 在你每次改完 Linux 内核源代码之后 - 试着编译它。我指的是任何修改之后，都要不断的编译。没有人喜欢那些连编译都不通过修改。

* Linux 内核有一套代码规范[指南](https://github.com/torvalds/linux/blob/master/Documentation/CodingStyle)，你需要遵守它。有一个很棒的脚本可以帮你检查所做的修改。这个脚本就是 - [scripts/checkpatch.pl](https://github.com/torvalds/linux/blob/master/scripts/checkpatch.pl)。只需要将被改动的源码文件传递给它即可，然后你就会看到如下输出：

```
$ ./scripts/checkpatch.pl -f drivers/staging/dgap/dgap.c
WARNING: Block comments use * on subsequent lines
#94: FILE: drivers/staging/dgap/dgap.c:94:
+/*
+     SUPPORTED PRODUCTS

CHECK: spaces preferred around that '|' (ctx:VxV)
#143: FILE: drivers/staging/dgap/dgap.c:143:
+	{ PPCM,        PCI_DEV_XEM_NAME,     64, (T_PCXM|T_PCLITE|T_PCIBUS) },

```

在 `git diff` 命令的帮助下，你也会看到一些有问题的地方：

![git diff](http://oi60.tinypic.com/2u91rgn.jpg)

* [Linus 不接受 github pull requests](https://github.com/torvalds/linux/pull/17#issuecomment-5654674)

* 如果你的修改是由一些不同的且不相关的改动所组成的，你需要通过分离提交来切分修改。`git format-patch` 命令将会为每个提交生成一个补丁，每个补丁的标题会包含一个 `vN` 前缀，其中 `N` 是补丁的编号。如果你打算发送一系列补丁，也许给 `git format-patch` 命令传递 `--cover-letter` 选项会对此很有帮助。这会生成一个附加文件，该文件包括的附函可以用来描述你的补丁集所做的改动。在 `git send-email` 命令中使用 `--in-reply-to` 选项也是一个好主意，该选项允许你将补丁集作为对附函的回复发送出去。对于维护者来说，你补丁集的结构看起来就像下面这样：

```
|--> cover letter
  |----> patch_1
  |----> patch_2
```

你可以将 `message-id` 参数传递给 `--in-reply-to` 选项，该选项可以在 `git send-email` 命令的输出中找到。

有一件非常重要的事，那就是你的邮件必须是[纯文本](https://en.wikipedia.org/wiki/Plain_text)格式。通常来说，`send-email` 和 `format-patch` 这两个命令在内核开发中都是非常有用的，所以请查阅这些命令的的相关文档，你会发现很多有用的选项，例如：[git send-email](http://git-scm.com/docs/git-send-email) 和 [git format-patch](http://git-scm.com/docs/git-format-patch)。

* 如果你发完补丁之后没有得到立即答复，请不要惊讶，因为维护者们都是很忙的。

* [scripts](https://github.com/torvalds/linux/tree/master/scripts) 目录包含了很多对 Linux 内核开发有用的脚本。我们已经看过此目录中的两个脚本了：`checkpatch.pl` 和 `get_maintainer.pl`。除此之外，你还可以找到 [stackusage](https://github.com/torvalds/linux/blob/master/scripts/stackusage) 脚本，它可以打印栈的使用情况，[extract-vmlinux](https://github.com/torvalds/linux/blob/master/scripts/extract-vmlinux) 脚本可以提取出未经压缩的内镜镜像，还有很多其他的脚本。在 `scripts` 目录之外，你也会发现很多有用的[脚本](https://github.com/lorenzo-stoakes/kernel-scripts)，这些脚本是 [Lorenzo Stoakes](https://twitter.com/ljsloz) 为内核开发而编写的。

* 订阅 Linux 内核邮件列表。`lkml` 列表中每天都会有大量的信件，但是阅读它们并了解一些类似于 Linux 内核目前开发状态的内容是很有帮助的。除了 `lkml` 之外，还有[一些](http://vger.kernel.org/vger-lists.html)其他的邮件列表，它们分别对应于不同的 Linux 内核子系统。

* 如果你发的补丁第一次没有被接受，你就会收到 Linux 内核开发者的反馈。请做一些修改然后以 `[PATCH vN]`(`N` 是补丁版本号) 为前缀重新发送补丁，例如：

```
[PATCH v2] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

同样的，这次的补丁也必须包括更新日志以便描述自上一次的补丁以来所做的修改。当然，本文并不是对 Linux 内核开发详尽无遗的指导清单，但是一些最重要的事项已经都被阐明了。

Happy Hacking!

总结
--------------------------------------------------------------------------------

我希望这篇文章能够帮助其他人加入 Linux 内核社区！
如果你有其他问题或建议，可以给我写[邮件](kuleshovmail@gmail.com)或者在 Twitter 上联系[我](https://twitter.com/0xAX)。

请注意，英语并不是我的母语，对此带来的不便我感到很抱歉。如果你发现了错误，请通过邮件或发 PR 来通知我。

相关链接
--------------------------------------------------------------------------------

* [blog posts about assembly programming for x86_64](http://0xax.github.io/categories/assembly/)
* [Assembler](https://en.wikipedia.org/wiki/Assembly_language#Assembler)
* [distro](https://en.wikipedia.org/wiki/Linux_distribution)
* [package manager](https://en.wikipedia.org/wiki/Package_manager)
* [grub](https://en.wikipedia.org/wiki/GNU_GRUB)
* [kernel.org](https://kernel.org/)
* [version control system](https://en.wikipedia.org/wiki/Version_control)
* [arm64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features)
* [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)
* [qemu](https://en.wikipedia.org/wiki/QEMU)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [busybox](https://en.wikipedia.org/wiki/BusyBox)
* [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [Linux kernel mail listing archive](https://lkml.org/)
* [Linux kernel coding style guide](https://github.com/torvalds/linux/blob/master/Documentation/CodingStyle)
* [How to Get Your Change Into the Linux Kernel](https://github.com/torvalds/linux/blob/master/Documentation/SubmittingPatches)
* [Linux Kernel Newbies](http://kernelnewbies.org/)
* [plain text](https://en.wikipedia.org/wiki/Plain_text)
