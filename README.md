# Linux 内核揭秘

一系列关于 Linux 内核和其内在机理的帖子。

**目的很简单** - 分享我对 Linux 内核机理的一些浅见，帮助读者理解 Linux 内核机理和其他底层内容。

**问题/建议**: 如有相关问题或建议，请提交 issue。一方面，对于英文原文问题，请在上游仓库 - [linux-insides](https://github.com/0xAX/linux-insides) 中提交 issue；另一方面，对于中文翻译问题，请在下游仓库 - [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 中提交 issue。

## 贡献

如有相关问题或建议，请不吝指教，提交 issues 或者 PRs。对于 `linux-insides-zh` 翻译项目，请通过以下方法进行贡献：

- 英文翻译，目前只提供简体中文的译文；
- 更新未被翻译的英文原本，其实就是将上游英文的更新纳入到当前项目中；
- 更新已经翻译的中文译文，其实就是查看上游英文的更新，检查是否需要对中文译文进行更新；
- 校对当前已经翻译过的中文译文，包括修改错别字，润色等工作；

在开始翻译之前，请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 与 [TRANSLATION_NOTES.md](TRANSLATION_NOTES.md)。关于翻译约定，大家有任何问题或建议也请提交 issue 讨论。

## 翻译

### 翻译进度展示

| 章节|译者|翻译进度|
| ------------- |:-------------:| -----:|
| 1. [Booting](https://github.com/MintCN/linux-insides-zh/tree/master/Booting)||已完成|
|├ [1.0](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/README.md)|[@xinqiu](https://github.com/xinqiu)|更新至[527b2b8](https://github.com/0xAX/linux-insides/commit/527b2b8921c3d9c043bd914c5990d6a991e3035b)|
|├ [1.1](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-1.md)|[@hailincai](https://github.com/hailincai)|已完成|
|├ [1.2](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-2.md)|[@hailincai](https://github.com/hailincai)|已完成|
|├ [1.3](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-3.md)|[@hailincai](https://github.com/hailincai)|已完成|
|├ [1.4](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-4.md)|[@zmj1316](https://github.com/zmj1316)|已完成|
|├ [1.5](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-5.md)|[@mytbk](https://github.com/mytbk)|更新至[31998d14](https://github.com/0xAX/linux-insides/commit/31998d14320f25399d67d4fff446a65178931e90)|
|└ [1.6](https://github.com/MintCN/linux-insides-zh/blob/master/Booting/linux-bootstrap-6.md)|[@mytbk](https://github.com/mytbk)|更新至[31998d14](https://github.com/0xAX/linux-insides/commit/31998d14320f25399d67d4fff446a65178931e90)|
| 2. [Initialization](https://github.com/MintCN/linux-insides-zh/tree/master/Initialization)||正在进行|
|├ [2.0](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[44017507](https://github.com/0xAX/linux-insides/commit/4401750766f7150dcd16f579026f5554541a6ab9)|
|├ [2.1](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-1.md)|[@dontpanic92](https://github.com/dontpanic92)|更新至[44017507](https://github.com/0xAX/linux-insides/commit/4401750766f7150dcd16f579026f5554541a6ab9)|
|├ [2.2](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-2.md)|[@dontpanic92](https://github.com/dontpanic92)|更新至[44017507](https://github.com/0xAX/linux-insides/commit/4401750766f7150dcd16f579026f5554541a6ab9)|
|├ [2.3](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-3.md)|[@dontpanic92](https://github.com/dontpanic92)|更新至[44017507](https://github.com/0xAX/linux-insides/commit/4401750766f7150dcd16f579026f5554541a6ab9)|
|├ [2.4](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-4.md)|[@bjwrkj](https://github.com/bjwrkj)|已完成|
|├ [2.5](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-5.md)|[@NeoCui](https://github.com/NeoCui)|更新至[cf32dc6c](https://github.com/0xAX/linux-insides/commit/cf32dc6c81abce567af330c480afc3d58678443d)|
|├ [2.6](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-6.md)|[@kele1997](https://github.com/kele1997)|更新至[e896e56](https://github.com/0xAX/linux-insides/commit/e896e56c867876397ef78da58d5e2a31b2e690b6)|
|├ [2.7](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-7.md)||未开始|
|├ [2.8](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-8.md)||未开始|
|├ [2.9](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-9.md)||未开始|
|└ [2.10](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-10.md)||未开始|
| 3. [Interrupts](https://github.com/MintCN/linux-insides-zh/tree/master/Interrupts)||正在进行|
|├ [3.0](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/README.md)|[@littleneko](https://github.com/littleneko)|更新至[57279321](https://github.com/0xAX/linux-insides/commit/5727932167a2ff6a1e647081c85d081d4ed8b508)|
|├ [3.1](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-1.md)|[@Albertchamberlain](https://github.com/Albertchamberlain)|更新至[e58c06bf](https://github.com/0xAX/linux-insides/commit/e58c06bfca60d4af25d92562de1ee9959992fc68)|
|├ [3.2](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-2.md)|[@narcijie](https://github.com/narcijie)|更新至[4d635117](https://github.com/0xAX/linux-insides/commit/4d6351172486e5c046a7d3db2286fc0d0d0d7789)|
|├ [3.3](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-3.md)||未开始|
|├ [3.4](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-4.md)||未开始|
|├ [3.5](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-5.md)||未开始|
|├ [3.6](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-6.md)||未开始|
|├ [3.7](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-7.md)||未开始|
|├ [3.8](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-8.md)||未开始|
|├ [3.9](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-9.md)|[@zhangyangjing](https://github.com/zhangyangjing)|已完成|
|└ [3.10](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/linux-interrupts-10.md)|[@worldwar](https://github.com/worldwar)|已完成|
| 4. [System calls](https://github.com/MintCN/linux-insides-zh/tree/master/SysCall)||正在进行|
|├ [4.0](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[194d0c83](https://github.com/0xAX/linux-insides/commit/194d0c83e3273c6167830c29d9ba13ec57bfbcb6)|
|├ [4.1](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-1.md)|[@qianmoke](https://github.com/qianmoke)|已完成|
|├ [4.2](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-2.md)|[@qianmoke](https://github.com/qianmoke)|已完成|
|├ [4.3](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-3.md)||未开始|
|├ [4.4](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-4.md)||未开始|
|├ [4.5](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-5.md)|[@asanzjx](https://github.com/asanzjx)|已完成|
|└ [4.6](https://github.com/MintCN/linux-insides-zh/blob/master/SysCall/linux-syscall-6.md)||未开始|
| 5. [Timers and time management](https://github.com/MintCN/linux-insides-zh/tree/master/Timers)||正在进行|
|├ [5.0](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[2a742fd4](https://github.com/0xAX/linux-insides/commit/2a742fd485df0260efce2078e7162c0de668e98b)|
|├ [5.1](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-1.md)||未开始|
|├ [5.2](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-2.md)||未开始|
|├ [5.3](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-3.md)||未开始|
|├ [5.4](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-4.md)||未开始|
|├ [5.5](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-5.md)||未开始|
|├ [5.6](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-6.md)||未开始|
|└ [5.7](https://github.com/MintCN/linux-insides-zh/blob/master/Timers/linux-timers-7.md)||未开始|
| 6. [Synchronization primitives](https://github.com/MintCN/linux-insides-zh/tree/master/SyncPrim)||正在进行|
|├ [6.0](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[6f85b63e](https://github.com/0xAX/linux-insides/commit/6f85b63e347b636e08e965e9dc22c177e972afe2)|
|├ [6.1](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-1.md)|[@keltoy](https://github.com/keltoy)|已完成|
|├ [6.2](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-2.md)|[@keltoy](https://github.com/keltoy)|已完成|
|├ [6.3](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-3.md)|[@huxq](https://github.com/huxq)|已完成|
|├ [6.4](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-4.md)|[@nannxnann](https://github.com/nannxnann)|更新至[c4b17d17](https://github.com/0xAX/linux-insides/commit/c4b17d17a085608e8de6e310797d8e81927aed8d)|
|├ [6.5](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-5.md)||未开始|
|└ [6.6](https://github.com/MintCN/linux-insides-zh/blob/master/SyncPrim/linux-sync-6.md)||未开始|
| 7. [Memory management](https://github.com/MintCN/linux-insides-zh/tree/master/MM)||未开始|
|├ [7.0](https://github.com/MintCN/linux-insides-zh/blob/master/MM/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[f83c8ee2](https://github.com/0xAX/linux-insides/commit/f83c8ee29e2051a8f4c08d6a0fa8247d934e14d9)|
|├ [7.1](https://github.com/MintCN/linux-insides-zh/blob/master/MM/linux-mm-1.md)|[@choleraehyq](https://github.com/choleraehyq)|已完成|
|├ [7.2](https://github.com/MintCN/linux-insides-zh/blob/master/MM/linux-mm-2.md)|[@choleraehyq](https://github.com/choleraehyq)|已完成|
|└ [7.3](https://github.com/MintCN/linux-insides-zh/blob/master/MM/linux-mm-3.md)|[@lifangwang](https://github.com/lifangwang)|已完成|
| 8. SMP||上游未开始|
| 9. [Concepts](https://github.com/MintCN/linux-insides-zh/tree/master/Concepts)||已完成|
|├ [9.0](https://github.com/MintCN/linux-insides-zh/blob/master/Concepts/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[44017507](https://github.com/0xAX/linux-insides/commit/4401750766f7150dcd16f579026f5554541a6ab9)|
|├ [9.1](https://github.com/MintCN/linux-insides-zh/blob/master/Concepts/linux-cpu-1.md)|[@up2wing](https://github.com/up2wing)|更新至[28a39fe6](https://github.com/0xAX/linux-insides/commit/28a39fe6653e780641e80ab6e37c79ffafca07b0#diff-0460583622f03a52d7693094d6fa2452)|
|├ [9.2](https://github.com/MintCN/linux-insides-zh/blob/master/Concepts/linux-cpu-2.md)|[@up2wing](https://github.com/up2wing)|更新至[28a39fe6](https://github.com/0xAX/linux-insides/commit/28a39fe6653e780641e80ab6e37c79ffafca07b0#diff-0460583622f03a52d7693094d6fa2452)|
|├ [9.3](https://github.com/MintCN/linux-insides-zh/blob/master/Concepts/linux-cpu-3.md)|[@up2wing](https://github.com/up2wing)|更新至[28a39fe6](https://github.com/0xAX/linux-insides/commit/28a39fe6653e780641e80ab6e37c79ffafca07b0#diff-0460583622f03a52d7693094d6fa2452)|
|└ [9.4](https://github.com/MintCN/linux-insides-zh/blob/master/Concepts/linux-cpu-4.md)||未开始|
| 10. [DataStructures](https://github.com/MintCN/linux-insides-zh/tree/master/DataStructures)||已完成|
|├ [10.0](https://github.com/MintCN/linux-insides-zh/blob/master/DataStructures/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[99138e09](https://github.com/0xAX/linux-insides/commit/99138e0932dc25bf6c90dd102a70a6d15589e9ab)|
|├ [10.1](https://github.com/MintCN/linux-insides-zh/blob/master/DataStructures/linux-datastructures-1.md)|[@oska874](http://github.com/oska874) [@mudongliang](https://github.com/mudongliang)|已完成|
|├ [10.2](https://github.com/MintCN/linux-insides-zh/blob/master/DataStructures/linux-datastructures-2.md)|[@a1ickgu0](https://github.com/a1ickgu0)|已完成|
|└ [10.3](https://github.com/MintCN/linux-insides-zh/blob/master/DataStructures/linux-datastructures-3.md)|[@cposture](https://github.com/cposture)|已完成|
| 11. [Theory](https://github.com/MintCN/linux-insides-zh/tree/master/Theory)||正在进行|
|├ [11.0](https://github.com/MintCN/linux-insides-zh/blob/master/Theory/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[99ad0799](https://github.com/0xAX/linux-insides/commit/99ad07999636b76985218e02e5a52140050cbbde)|
|├ [11.1](https://github.com/MintCN/linux-insides-zh/blob/master/Theory/linux-theory-1.md)|[@mudongliang](https://github.com/mudongliang)|已完成|
|├ [11.2](https://github.com/MintCN/linux-insides-zh/blob/master/Theory/linux-theory-2.md)|[@mudongliang](https://github.com/mudongliang)|已完成|
|└ [11.3](https://github.com/MintCN/linux-insides-zh/blob/master/Theory/linux-theory-3.md)||未开始|
| 12. Initial ram disk||上游未开始|
| 13. [Misc](https://github.com/MintCN/linux-insides-zh/tree/master/Misc)||已完成|
|├ [13.0](https://github.com/MintCN/linux-insides-zh/blob/master/Misc/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[ddf0793f](https://github.com/0xAX/linux-insides/commit/ddf0793f6b74b6c541c4979b4deaf093b2b87c9b)|
|├ [13.1](https://github.com/MintCN/linux-insides-zh/blob/master/Misc/linux-misc-1.md)|[@hao-lee](https://github.com/hao-lee)|更新至[3ed52146](https://github.com/0xAX/linux-insides/commit/3ed521464e99a8ff2f8d438592a605a716a268e2)|
|├ [13.2](https://github.com/MintCN/linux-insides-zh/blob/master/Misc/linux-misc-2.md)|[@oska874](https://github.com/oska874)|已完成|
|├ [13.3](https://github.com/MintCN/linux-insides-zh/blob/master/Misc/linux-misc-3.md)|[@zmj1316](https://github.com/zmj1316)|已完成|
|└ [13.4](https://github.com/MintCN/linux-insides-zh/blob/master/Misc/linux-misc-4.md)|[@mudongliang](https://github.com/mudongliang)|已完成|
| 14. [KernelStructures](https://github.com/MintCN/linux-insides-zh/tree/master/KernelStructures)||已完成|
|├ [14.0](https://github.com/MintCN/linux-insides-zh/tree/master/KernelStructures/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[3cb550c0](https://github.com/0xAX/linux-insides/commit/3cb550c089c8fc609f667290434e9e98e93fa279)|
|└ [14.1](https://github.com/MintCN/linux-insides-zh/tree/master/KernelStructures/linux-kernelstructure-1.md)|[@woodpenker](https://github.com/woodpenker)|更新至[4521637d](https://github.com/0xAX/linux-insides/commit/4521637d9cb76e5d4e4dc951758b264a68504927)|
| 15. [Cgroups](https://github.com/MintCN/linux-insides-zh/tree/master/Cgroups)||已完成|
|├ [15.0](https://github.com/MintCN/linux-insides-zh/tree/master/Cgroups/README.md)|[@mudongliang](https://github.com/mudongliang)|更新至[e811ca4f](https://github.com/0xAX/linux-insides/commit/90f50c2ac5a197da044e5091c631dd43e811ca4f)|
|└ [15.1](https://github.com/MintCN/linux-insides-zh/tree/master/Cgroups/linux-cgroups-1.md)|[@tjm-1990](https://github.com/tjm-1990)|更新至[b420e581](https://github.com/0xAX/linux-insides/commit/b420e581fe3cfee64d9c65103740d4fd98127b6f)|

### 翻译认领规则

为了避免多个译者同时翻译相同章节的情况出现，请按照以下规则认领自己要翻译的章节：

* 在 [README.md](README.md) 中查看你想翻译的章节的状态；
* 在确认想翻译的章节没有被翻译之后，开一个 issue ，告诉大家你想翻译哪一章节，同时提交申请翻译的 PR ，将 [README.md](https://github.com/MintCN/linux-insides-zh/blob/master/README.md) 中的翻译状态修改为“正在进行”；
* 首先，从上游的[英文库](https://github.com/0xAX/linux-insides)中得到该章节的最新版本，将修改提交到我们的中文库中；
* 然后翻译你认领的章节；
* 完成翻译之后，提交翻译内容的 PR (**注：大家最好以一个文件为基本单位来提交翻译 PR，方便我们进行 review，否则可能会因为 comments 导致展示 PR 的网页变得过于冗长，不方便 review 修改的内容**)。待 PR 被合并之后，请关闭 issue；
* 最后，将 [README.md](README.md) 中的翻译状态修改为“更新至上游 commit id”(**注：8 位 commit id**)，同时不要忘记把自己添加到 [CONTRIBUTORS.md](CONTRIBUTORS.md) 中。

### 翻译回收原则

为了避免翻译申请过长时间没有任何更新，所以暂设置时间为三个月，如果三个月之内，issue 并没有任何更新，翻译申请就会被回收。

## 原作者

[@0xAX](https://twitter.com/0xAX)

## 中文维护者

[@xinqiu](https://github.com/xinqiu)

[@mudongliang](https://github.com/mudongliang)

## 中文贡献者

详见 [CONTRIBUTORS.md](CONTRIBUTORS.md)

## LICENSE

Licensed [BY-NC-SA Creative Commons](http://creativecommons.org/licenses/by-nc-sa/4.0/).
