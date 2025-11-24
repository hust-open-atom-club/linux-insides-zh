# linux_patch_learn

patch review and learn

## roadmap

- [The state of the page in 2025（LSFMM 2025）](https://zhuanlan.zhihu.com/p/1889756066042082709)
  - 在 2025 年，目标是让 struct folio 确实成为一个与 struct page 分离的结构，并且可以独立分配。然后，可以从 struct page 中移除一些数据，缩小它，但还不能完全移除。
- [MatthewWilcox/Memdescs/Path - Linux Kernel Newbies](https://kernelnewbies.org/MatthewWilcox/Memdescs/Path)
- [MatthewWilcox/BuddyAllocator - Linux Kernel Newbies](https://kernelnewbies.org/MatthewWilcox/BuddyAllocator)

## THP

- 2010-11-03 [\[PATCH 00 of 66\] Transparent Hugepage Support #32 - Andrea Arcangeli](https://lore.kernel.org/all/patchbomb.1288798055@v2.random/)
  - 支持 anon THP
  - v33 https://lore.kernel.org/all/20101215051540.GP5638@random.random/
  - thp: transparent hugepage core
    - 处理 anon page fault 时，会预先分配好一个 PTE pagetable，存放到 mm_struct 粒度的链表里。现在这个函数叫做 `pgtable_trans_huge_deposit()`，与之相对应的函数是 `pgtable_trans_huge_withdraw()`，即存款和提款。
    - zap_huge_pmd() 时，会把这个预留的 pagetalbe 释放掉。
- 2014-11-11 [Transparent huge page reference counting \[LWN.net\]](https://lwn.net/Articles/619738/)
- 2015-10-06 [\[PATCHv12 00/37\] THP refcounting redesign - Kirill A. Shutemov](https://lore.kernel.org/linux-mm/1444145044-72349-1-git-send-email-kirill.shutemov@linux.intel.com/)
  - 新的 refcount mapcout 方案
    - anon THP 同时存在 PMD map 和 PTE map 时，会给所有 subpage 的 mapcount +1，这是为了保证 atomici page_remove_rmap()；并且，还会加上 PG_double_map bit，用于在 page_remove_rmap() 时判断是否同时存在 anon THP 的 PMD map 和 PTE map，如果同时存在，并且此时正在 remove 最后一个 PMD map 了，就需要把之前给所有 subpage +1 的 mapcount 给 -1 回来。
  - 支持 THP 的 PMD map 和 PTE map 共存
  - [PATCHv12 29/37] thp: implement split_huge_pmd() 新的 PMD 页表拆分实现
    - 会 page_ref_add(page, HPAGE_PMD_NR - 1); 这是因为多出了 512 个 PTE 映射，少了 1 个 PMD 映射，而对 subpage 进行 get_page() 实际上是对 head page 操作的。
  - [PATCHv12 30/37] thp: add option to setup migration entries during PMD split
    1. [PATCH RFC 和之前一样依赖于 compound_lock()](https://lore.kernel.org/linux-mm/1402329861-7037-7-git-send-email-kirill.shutemov@linux.intel.com/)
    2. [从 PATCHv2 开始](https://lore.kernel.org/linux-mm/1415198994-15252-19-git-send-email-kirill.shutemov@linux.intel.com/)，则是通过 migration PTE entries 来 stabilize page counts，也就是把页面放进 swapcache？和 try_to_unmap 差不多。
  - [PATCHv12 32/37] thp: reintroduce split_huge_page() 新的 THP 大页拆分实现
    1. 持有 anon_vma 锁，因为接下来我们要 rmap walk 了
    2. 检查是不是只有 caller 有额外的一个 refcount（也就是除了与 mapcount 一一对应的 refcount 以外，还有其他的 refcount，这也意味着现在页面被 pin 住了无法 migrate）
    3. `freeze_page()`：这个函数名不够好，其实就是反向映射，并做页表拆分
    4. 遍历 anon_vma 区间树，找到所有映射了该大页的 PMD 虚拟地址
    5. `freeze_page_vma()` 拆分 PMD 页表。有可能已经 swap out 了，页表已经拆分了，这时则是处理这些 PTE swap entry。
  - [PATCHv12 34/37] thp: introduce deferred_split_huge_page() 首次支持延迟拆分大页。如果某个 THP 已经不存在 PMD map，如果其中某些 subpage 不存在 PTE map，那么这些 subpage 也许是可以被释放的（之所以说“也许”，是因为还要考虑到 refcount），这就需要先 split THP 拆成小页，然后才能释放。这个 patch 做的事情：在 subpage 也许可以被释放时，把要拆分的 THP 放进一个队列，等内存回收时由 shrinker 来释放。
    - 在 page_remove_rmap() PMD page 时，如果这是最后一个 unmap 的大页，并且有 nr 个 subpage 没有 PTE map，说明这 nr 个 subpage 可以被释放，把 THP 放进队列。
    - 在 page_remove_rmap() subpage 时，如果 unmap 该 subpage 后，该 subpage 的 mapcount 为 -1，这说明，首先，已经没有 PageDoubleMap 带来的 1 个 mapcount，即，该 THP 没有 PMD map 了，另外，还说明该 subpage 没有 PTE map 了。于是把 THP 放进队列。
    - 定义了一个 deferred_split_shrinker
    - 在拆分 THP 时，如果该大页在队列内，则将其从队列中移除。
    - [ ] 对 mlocked THP 的处理
- 2016-03-07 [\[PATCHv2 0/4\] thp: simplify freeze_page() and unfreeze_page() - Kirill A. Shutemov](https://lore.kernel.org/linux-mm/1457351838-114702-1-git-send-email-kirill.shutemov@linux.intel.com/)
  - 在大页拆分时，使用通用的 rmap walker `try_to_unmap()`，简化了 `freeze_page()` 和 `unfreeze_page()`
    - try_to_unmap() 见 https://www.cnblogs.com/tolimit/p/5432674.html
  - TTU_SPLIT_HUGE_PMD 会让 try_to_unmap() 时先 split_huge_pmd_address() 拆分 PMD 页表。注意每次调用 try_to_unmap() 只会 unmap 一个 page 的所有反向映射，所以要调用 HPAGE_PMD_NR 次。
- 2016-05-11 [Transparent huge pages in the page cache \[LWN.net\]](https://lwn.net/Articles/686690/)
- 2016-06-15 [\[PATCHv9 00/32\] THP-enabled tmpfs/shmem using compound pages - Kirill A. Shutemov](https://lore.kernel.org/linux-mm/1465222029-45942-1-git-send-email-kirill.shutemov@linux.intel.com/)
  - 支持 tmpfs/shmem THP
  - [PATCHv9 05/32] rmap: support file thp
    - [ ] `page_add_file_rmap()` 对于 THP 会把每个 subpage 的 mapcount 都 +1。不理解为什不能和 `page_add_anon_rmap()` 一样，commit message 里说是后续再优化。
    - [ ] 不理解。PG_double_map 的优化对 file page 无效，这是因为 lifecycle 与 anon page 不同，file page 在没有 map 时还可以继续存在，随时再次被 map。
  - thp: support file pages in zap_huge_pmd()
  - thp: handle file pages in split_huge_pmd()
    - 只做了 unmap，没有像 anon page 那样分配页表去填 PTE，因为 file page 可以等到 page fault 时再去填 PTE 页表。不理解，如果填 PTE 页表，避免后续可能的 pagefault 不是很好吗？
  - thp: handle file COW faults
    - split huge pmd 然后在 pte level 处理。因为不清楚在 private file page CoW 场景分配 huge page 的收益如何，可能是过度设计。
  - thp: skip file huge pmd on copy_huge_pmd()
    - 典型场景：进程 clone。对于 file pages，可以不 alloc pagetable，不 copy pte/pmd，可以在 pagefault 时做。copy_huge_pmd() 的调用路径只有 copy_page_range()，后者会使得没有 vma->anon_vma 的跳过 copy pte/pmd。但是因为 private file mapping 是可以有 anon_vma 的，所以没有跳过，这里选择了让 copy_huge_pmd() 通过 vma->vm_ops 把这种情况检查出来，跳过 private file huge pmd 的 copy。
  - thp: file pages support for split_huge_page()
  - vmscan: split file huge pages before paging them out
  - filemap: prepare find and delete operations for huge pages
  - shmem: add huge pages support
- 2022-11-03 [\[PATCH 0/3\] mm,huge,rmap: unify and speed up compound mapcounts - Hugh Dickins](https://lore.kernel.org/linux-mm/5f52de70-975-e94f-f141-543765736181@google.com/)
  - 优化 compound mapcount
  - mm,thp,rmap: simplify compound page mapcount handling
- 2022-11-22 [\[PATCH v2 0/3\] mm,thp,rmap: rework the use of subpages_mapcount - Hugh Dickins](https://lore.kernel.org/linux-mm/a5849eca-22f1-3517-bf29-95d982242742@google.com/)
- 2024-04-09 [\[PATCH v1 00/18\] mm: mapcount for large folios + page_mapcount() cleanups - David Hildenbrand](https://lore.kernel.org/linux-mm/20240409192301.907377-1-david@redhat.com/)
- 2023-07-10 [\[PATCH v4 0/9\] Create large folios in iomap buffered write path - Matthew Wilcox (Oracle)](https://lore.kernel.org/linux-fsdevel/20230710130253.3484695-1-willy@infradead.org/)
- 2024-04-15 [\[PATCH v3 0/4\] mm/filemap: optimize folio adding and splitting - Kairui Song](https://lore.kernel.org/all/20240415171857.19244-1-ryncsn@gmail.com/)
- 2024-05-21 [Facing down mapcount madness \[LWN.net\]](https://lwn.net/Articles/974223/)
- 2024-02-26 [\[PATCH v5 0/8\] Split a folio to any lower order folios - Zi Yan](https://lore.kernel.org/linux-mm/20240226205534.1603748-1-zi.yan@sent.com/)
  - 支持将 folio split 到任意 low order
- 2025-03-07 [\[PATCH v10 0/8\] Buddy allocator like (or non-uniform) folio split - Zi Yan](https://lore.kernel.org/linux-mm/20250307174001.242794-1-ziy@nvidia.com/)
  - 支持 non-uniform folio split
- 2025-05-12 [\[PATCH v2 0/8\] ext4: enable large folio for regular files - Zhang Yi](https://lore.kernel.org/all/20250512063319.3539411-1-yi.zhang@huaweicloud.com/)
  - 为 ext4 regular files 支持 large folio
- 2017-05-15 🚧 [\[PATCH -mm -v11 0/5\] THP swap: Delay splitting THP during swapping out - Huang, Ying](https://lore.kernel.org/linux-mm/20170515112522.32457-1-ying.huang@intel.com/)


selftest

- 2025-8-18[\[PATCH v5 0/5\] Better split_huge_page_test result check](https://lore.kernel.org/linux-mm/20250818184622.1521620-1-ziy@nvidia.com/)
这一组patch增加了对于thp分裂后的order检查，Just note that the code does not handle memremapped THP, since
it only checks page flags without checking the PFN. So when a vaddr range is mapped
to a THP/mTHP head page and some other THP/mTHP tail pages, the code just treats
the whole vaddr range as if it is mapped to a single THP/mTHP and gets a wrong
order. After-split folios do not have this concern, so
gather_after_split_folio_orders() is simplified to not handle such cases.
目前支持的场景如上，虽然baoling老师重用了这组patch在[\[RFC PATCH 00/11\] add shmem mTHP collapse support](https://lore.kernel.org/all/955e0b9682b1746c528a043f0ca530b54ee22536.1755677674.git.baolin.wang@linux.alibaba.com/)但是可能有点仍会出现问题，目前ziyan老师局限的这种场景比较稳健

TAO

- 2024-02-29 🚧 [\[LSF/MM/BPF TOPIC\] TAO: THP Allocator Optimizations - Yu Zhao](https://lore.kernel.org/linux-mm/20240229183436.4110845-1-yuzhao@google.com/)
- 2024-05-24 [Allocator optimizations for transparent huge pages \[LWN.net\]](https://lwn.net/Articles/974636/)

## mTHP

- 2023-12-07 [\[PATCH v9 00/10\] Multi-size THP for anonymous memory - Ryan Roberts](https://lore.kernel.org/linux-mm/20231207161211.2374093-1-ryan.roberts@arm.com/)
- 2024-09-20 [Linux Plumbers Conference 2024: Product practices of large folios on millions of OPPO Android phones](https://lpc.events/event/18/contributions/1705/)
- 2025-08-14 [\[RFC PATCH 0/7\] add mTHP support for wp - Vernon Yang](https://lore.kernel.org/linux-mm/20250814113813.4533-1-vernon2gm@gmail.com/)
- 2025-08-19 [\[PATCH v10 00/13\] khugepaged: mTHP support - Nico Pache](https://lore.kernel.org/linux-mm/20250819134205.622806-1-npache@redhat.com/)
- 2025-08-20 [\[RFC PATCH 00/11\] add shmem mTHP collapse support - Baolin Wang](https://lore.kernel.org/linux-mm/cover.1755677674.git.baolin.wang@linux.alibaba.com/)
- [An Empirical Evaluation of PTE Coalescing](https://www.eliot.so/memsys23.pdf)
- [Every Mapping Counts in Large Amounts: Folio Accounting](https://www.usenix.org/system/files/atc24-hildenbrand.pdf)

selftests

- 2025-08-18 [\[PATCH v5 0/5\] Better split_huge_page_test result check - Zi Yan](https://lore.kernel.org/linux-mm/20250818184622.1521620-1-ziy@nvidia.com/)

## CONT PTE

- 2024-02-15 [\[PATCH v6 00/18\] Transparent Contiguous PTEs for User Mappings - Ryan Roberts](https://lore.kernel.org/linux-mm/20240215103205.2607016-1-ryan.roberts@arm.com/)

## rmap

selftests

- 2025-08-19 [\[Patch v4 0/2\] test that rmap behaves as expected - Wei Yang](https://lore.kernel.org/all/20250819080047.10063-1-richard.weiyang@gmail.com/)

## madvise

- 2025-06-07 [\[PATCH v4\] mm: use per_vma lock for MADV_DONTNEED - Barry Song](https://lore.kernel.org/all/20250607220150.2980-1-21cnbao@gmail.com/)

  - mm: madvise: use walk_page_range_vma() instead of walk_page_range()

    - do_madvise [behavior=MADV_DONTNEED]

      - madvise_lock

        - lock_vma_under_rcu
          - madvise_do_behavior
            - madvise_single_locked_vma
              - madvise_vma_behavior
                - madvise_dontneed_free
                  - madvise_dontneed_single_vma
                    - map_page_range_single_batched [.reclaim_pt = true]
                      - unmap_single_vma
                        - unmap_page_range
                          - zap_p4d_range
                            - zap_pud_range
                              - zap_pmd_range
                                - zap_pte_range
                                  - try_get_and_clear_pmd
                                    - free_pte

        调用关系如上所示 do_behavior 中遍历会调用 madvise_walk_vmas 就已经进行了 vma 的查找，之后调用 madvise_free_single_vma 时就不需要在 walk_page_range 进行 vma 的查找了，直接使用 use walk_page_range_vma()传入 vma 参数就可以，减少了一次 vma 的查找开销

  - mm: use per_vma lock for MADV_DONTNEED
    目前支持的 per vma 仅限于本地进程 single vma 同时不能涉及 uffd,这样的情况使用 rcu 机制可以极大的降低优先级翻转和读者等待，其他的情况回退到 mmap_lock（读写锁）, 新的锁的模式 MADVISE_VMA_READ_LOCK 区别原来的读写锁只有 dontneed 和 free 这俩行为支持
  - mm: madvise: use per_vma lock for MADV_FREE
    为 free 扩展 per vma 支持，同时之前的 walk page 的路径中增加 PGWALK_VMA_RDLOCK_VERIFY 只会锁住当前的 vma
  - mm: fix the race between collapse and PT_RECLAIM under per-vma lock
    collapse 合并时操作的是整个的 2M 空间的 vma，而之前的 dontneed 和 free 的逻辑在回收时候允许支持 per vma 造成了 lock race，通过改变 lock 顺序解除 lock race

## msharefs

- 2025-08-20 [\[PATCH v3 00/22\] Add support for shared PTEs across processes - Anthony Yznaga](https://lore.kernel.org/linux-mm/20250820010415.699353-1-anthony.yznaga@oracle.com/)

## LUO

- [\[PATCH v3 00/30\] Live Update Orchestrator - Pasha Tatashin](https://lore.kernel.org/linux-mm/20250807014442.3829950-1-pasha.tatashin@soleen.com/)

##

- [Formalizing policy zones for memory \[LWN.net\]](https://lwn.net/Articles/964239/)

## mm init

- 2025-08-27 [\[PATCH v1 00/36\] mm: remove nth_page()](https://lore.kernel.org/linux-mm/20250827220141.262669-1-david@redhat.com/T/#mc904b4675c39f993fb43a0098637e087166d6df7)
 #define pfn_to_page(pfn) (void *)((pfn) * PAGE_SIZE)
初始化的重构非常的有意思，开始的平坦内存是初始化的page对应一个连续数组，但是这样会导致很大的内存浪费，后面引入了稀疏内存将内存分为section 一般64位普遍是128M对应一个section，这部分内存对应一个page的数组，便于pfn和page的转化，因为不连续所以需要这个#define nth_page(page,n) pfn_to_page(page_to_pfn((page)) + (n))来找到下一个数组里面对应的page，这样反复的计算非常的麻烦但是又不得不做，因为可能会跨section导致直接递增寻址失败，david强制使用vmemmap和加入判读避免buddy，hugetlb，cma等大块内存申请超越section的访问，后面我会更新文章记录下

## vma optimization
- 2023-02-07 [\[PATCH v4 00/33\] Per-VMA locks](https://lore.kernel.org/linux-mm/20250827220141.262669-1-david@redhat.com/T/#mc904b4675c39f993fb43a0098637e087166d6df7)
vma 减少锁的争用

## reclaim

- 2013-05-13 [\[PATCH 3/4\] mm: Activate !PageLRU pages on mark_page_accessed if page is on local pagevec - Mel Gorman](https://lore.kernel.org/linux-mm/1368440482-27909-4-git-send-email-mgorman@suse.de/)
- 2025-02-14 [\[PATCH v4 0/4\] mm: batched unmap lazyfree large folios during reclamation - Barry Song](https://lore.kernel.org/linux-mm/20250214093015.51024-1-21cnbao@gmail.com/)
- 2025-04-02 [\[PATCH v2 8/9\] mm: Remove swap_writepage() and shmem_writepage() - Matthew Wilcox (Oracle)](https://lore.kernel.org/all/20250402150005.2309458-9-willy@infradead.org/)
  在 shrink_folio_list 时，只有 shmem 和 anon 会 pageout，脏文件页不会 pageout

workingset

- 2014-02-04 [\[patch 00/10\] mm: thrash detection-based file cache sizing v9](https://lore.kernel.org/linux-mm/1391475222-1169-1-git-send-email-hannes@cmpxchg.org/)
  - 2012-05-02 [Better active/inactive list balancing \[LWN.net\]](https://lwn.net/Articles/495543/)
- 2019-11-07 [\[PATCH 0/3\] mm: fix page aging across multiple cgroups](https://lore.kernel.org/linux-mm/20191107205334.158354-1-hannes@cmpxchg.org/)
- 2020-05-20 [\[PATCH 00/14\] mm: balance LRU lists based on relative thrashing v2 - Johannes Weiner](https://lore.kernel.org/all/20200520232525.798933-1-hannes@cmpxchg.org/)
- 2020-07-23 [\[PATCH v7 0/6\] workingset protection/detection on the anonymous LRU list](https://lore.kernel.org/linux-mm/1595490560-15117-1-git-send-email-iamjoonsoo.kim@lge.com/)
  - 2020-03-10 [Working-set protection for anonymous pages \[LWN.net\]](https://lwn.net/Articles/815342/)

MGLRU

- 2022-09-18 [\[PATCH mm-unstable v15 00/14\] Multi-Gen LRU Framework - Yu Zhao](https://lore.kernel.org/linux-mm/20220918080010.2920238-1-yuzhao@google.com/)

## swap

- 2025-09-17  [\[PATCH v4 00/15\] mm, swap: introduce swap table as swap cache (phase I)- Kairui Song](https://lore.kernel.org/linux-mm/99f57a96-611a-b6be-fe00-3ac785154d1c@google.com/T/#m25856b1be7a11ec2bf1c482137244897541c7446)
  swap cache新的管理方式详情见这篇文章 swap学习记录和研究https://zhuanlan.zhihu.com/p/1911006969755578935

## Tiered Memory

- [PET: Proactive Demotion for Efficient Tiered Memory Management](https://dl.acm.org/doi/pdf/10.1145/3689031.3717471)
