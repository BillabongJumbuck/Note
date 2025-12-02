### 1. 路径

- 查看当前值

  `cat /proc/sys/vm/`

-  临时修改（立即生效，重启后失效）

  `sudo sysctl vm.swappiness=10`

-  永久修改

  - 编辑 `/etc/sysctl.conf` 文件

  ```
  sudo vim /etc/sysctl.conf
  ```

  - 在文件末尾添加或修改以下行：

  ```
  vm.swappiness = 10
  ```

  - 然后运行以下命令使配置立即生效：

  ```
  sudo sysctl -p
  ```



### 2. kswapd

​	**kswapd** 是 Linux 内核中一个非常重要的**守护进程 (daemon)**，负责管理系统的虚拟内存和物理内存之间的平衡。**kswapd** 是 Linux 内核中一个非常重要的**守护进程 (daemon)**，负责管理系统的虚拟内存和物理内存之间的平衡。

#### kswapd 的工作机制

#### 1. 内存阈值与唤醒

Linux 内核将物理内存划分为多个区域（zones），每个区域都设置了几个内存水位线（watermarks），通常是：

- **Min (最小):** 绝对底线，低于此水位系统进入紧急状态。
- **Low (低):** 唤醒 kswapd 开始工作的阈值。
- **High (高):** kswapd 停止工作的目标。

当一个内存区域的空闲页数量低于 **Low** 水位线时，**kswapd 就会被唤醒**。

#### 2. 内存回收 (Reclaiming)

kswapd 被唤醒后，它会开始回收内存，直到空闲页数量达到 **High** 水位线为止。它主要通过以下两种方式回收内存：

- **回收文件页 (File-backed pages):** 这主要是文件系统缓存（如 page cache）。如果这些缓存页没有被修改（即是干净的），kswapd 可以直接丢弃它们，因为数据还在磁盘上。
- **回收匿名页 (Anonymous pages):** 这主要是应用程序的私有数据（如堆、栈）。如果这些页是干净的，它们可以被直接释放；如果是脏的（被修改过），**kswapd 会将它们写入到交换空间（swap space）中**，这个过程就是常说的 **swapping**。

#### 什么时候会看到 kswapd 活跃？

如果使用 `top` 或 `htop` 等工具监控系统，发现 **kswapd0**（或 kswapd 加上区域编号，如 kswapd1）进程的 **CPU 使用率较高**时，这通常表明系统正在：

1. **物理内存不足**。
2. **正在进行频繁的内存回收操作**。
3. **频繁地将数据写入交换空间**。



### 3. 技术面指标

> 1、内存分配慢速路径次数降低xx％
>
> （整机性能测试或O测自动化测试）
>
> 约束条件：kswapd负载增加不超过5%。 
>
> 如果kswapd增加5％的话，慢速路径次数可能需要写降低10％。 如果k增加3％的话，慢速路径次数可能写6％

#### 1. 性能指标：内存分配慢速路径次数降低 xx%

- **内存分配慢速路径 (Slow Path Memory Allocation):**
  - 在 Linux 中，当应用程序请求内存时，内核会尝试通过“**快速路径 (Fast Path)**”立即满足请求。
  - 如果快速路径失败（即当前空闲内存不足），内核必须进入“**慢速路径 (Slow Path)**”。
  - 在慢速路径中，内核必须执行耗时的操作，例如：
    - 唤醒或等待 **kswapd** 来回收内存。
    - 执行直接回收（Direct Reclaim）——这会**阻塞（block）**当前的应用程序线程，直到内存被释放。
  - **结论：** **慢速路径次数越多，应用程序的延迟（lagging）和卡顿现象越严重。**
  - **目标：** 通过优化，将这些耗时的慢速路径次数降低一个百分比（如10%），意味着内存请求能更快地得到满足，提升整机性能。

#### 2. 约束条件：kswapd 负载增加不超过 5%

- **kswapd 负载：** 指的是 kswapd 守护进程所占用的 CPU 资源。kswapd 负责在后台进行内存回收和交换操作。
- **约束目的：** 优化内存分配（降低慢速路径）通常是通过让 **kswapd 更加积极地**在后台回收内存来实现的。然而，如果 kswapd 过于活跃，它会：
  - 占用大量的 CPU 资源，影响其他应用程序的运行。
  - 可能过于频繁地将数据换出到慢速交换空间，导致系统整体性能下降。
- **结论：** 这是一个**资源消耗的上限**。要求在提升应用程序响应速度的同时，不能通过过度消耗 CPU 资源（kswapd 负载）来达到目的。

#### 3. 量化关系：kswapd 增加 5% 对应慢速路径降低 10%

- 这个比例 (2:1) 设定了一个**性能/功耗比**的基线：**每多付出 1% 的 CPU 资源给 kswapd，至少要换来 2% 的内存分配效率提升。** 这样确保了优化是高效且有价值的.



### 4. 获取内存分配慢速路径次数这个指标

#### 1. 通过 `/proc/pagetypeinfo` 

#### 2. 通过 `/proc/vmstat`

| **指标名称**        | **含义**                                   | **与慢速路径的关系**                                    |
| ------------------- | ------------------------------------------ | ------------------------------------------------------- |
| `pgscan_direct`     | 应用程序线程主动扫描内存页进行回收的次数。 | **慢速路径的核心指标。** 此值越高，慢速路径发生越频繁。 |
| `pgscan_kswapd`     | kswapd 后台扫描内存页进行回收的次数。      | 这是后台正常（或优化后）的内存回收活动。                |
| `pgsteal_direct`    | 应用程序线程主动偷取/回收内存页的次数。    | 直接回收的结果，与 `pgscan_direct` 紧密相关。           |
| `pgsteal_kswapd`    | kswapd 后台回收内存页的次数。              | kswapd 的工作量。                                       |
| `kswapd_inodesteal` | kswapd 清理 inode/dentry 缓存的次数。      | 也是内存回收的一部分。                                  |

```SHELL
brumaire@DESKTOP-A9O5MKF ~> cat /proc/vmstat | grep pgscan_direct
pgscan_direct 0
pgscan_direct_throttle 0
```

#### 3. 监控 kswapd 负载



### 5. slowpath

```python
def slowpath_allocation():
    # 1. 降低标准 & 唤醒后台
    adjust_flags_to_use_reserves()
    wake_kswapd()
    if try_alloc(): return page

    # 2. 内存规整 (Defrag)
    if logic_for_compaction():
        compact_memory()
        if try_alloc(): return page

    # Loop start
    label retry:
    
    # 3. 直接回收 (最耗时的部分)
    if can_direct_reclaim:
        direct_reclaim() # <--- 阻塞当前进程，清理缓存
        if try_alloc(): return page
    
    # 4. 再次规整
    if logic_for_compaction():
        compact_memory()
        if try_alloc(): return page

    # 5. 决策：要不要重试？
    if should_retry(): goto retry

    # 6. OOM Killer (杀手锏)
    if trigger_oom():
        kill_process()
        if try_alloc(): return page

    # 7. 失败处理
    if is_nofail_flag_set(): goto retry # 死循环直到成功
    
    return NULL # 彻底失败
```





### 5. eBPF监测

#### 1. 可行性

**__alloc_pages_slowpath无法直接使用kprobe检测**

#### 2. 原始代码 `mm/page_alloc.c`

```c
/*
 * This is the 'heart' of the zoned buddy allocator.
 */
struct page *__alloc_pages(gfp_t gfp, unsigned int order, int preferred_nid,
							nodemask_t *nodemask)
{
	// ...

	/* First allocation attempt */
	page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);
	if (likely(page))
		goto out;

	alloc_gfp = gfp;
	ac.spread_dirty_pages = false;

	/*
	 * Restore the original nodemask if it was potentially replaced with
	 * &cpuset_current_mems_allowed to optimize the fast-path attempt.
	 */
	ac.nodemask = nodemask;

	page = __alloc_pages_slowpath(alloc_gfp, order, &ac);

out:
	// ...
    
	return page;
}
EXPORT_SYMBOL(__alloc_pages);
```

##### 因此可以严谨地得出结论????

> **一次慢速分配发生 ⇔ `get_page_from_freelist()` 返回 NULL**

这是唯一条件，没有其他路径能进 slowpath。

```
sudo bpftrace -e '
kretprobe:get_page_from_freelist
{
    if (retval == 0) {
        @slowpath_cnt++;
    }
}
'
```



这是一段非常经典的 Linux 内核代码，它是内存分配系统的**最后一道防线**，也是所有“慢速路径”逻辑的集大成者。

`__alloc_pages_slowpath` 的核心逻辑就是：**为了分配到内存，不惜一切代价（降低水位线、唤醒后台回收、同步直接回收、甚至杀掉进程）**。

为了方便您理解，我们可以将这个函数的工作流程分为几个**层层递进的阶段**。如果前一个阶段成功拿到了页 (`goto got_pg`)，函数就直接返回；否则，进入下一阶段。

------



### 阶段一：降低标准 & 呼叫支援 (Initial Retry)



当快速路径失败进入这里时，内核首先做的是“调整策略”和“呼叫后台支援”，而不是立即阻塞当前进程。

1. **呼叫后台支援 (`wake_all_kswapds`)**:
   - 既然内存不够了，赶紧唤醒后台的 `kswapd` 守护进程。让它在后台异步地回收内存，希望它能赶在当前进程彻底没内存之前释放出一些空间。
2. **降低水位线标准 (`gfp_to_alloc_flags`)**:
   - 快速路径使用的是比较严格的水位线（Watermark）。
   - 在这里，内核会修改 `alloc_flags`，允许分配器使用**更低的水位线**（min watermark），甚至允许访问为紧急情况保留的**内存储备（Reserves）**。
3. **再次尝试分配 (`get_page_from_freelist`)**:
   - 降低标准后，立刻尝试分配一次。如果运气好，利用储备内存就分配成功了，直接返回。



### 阶段二：内存规整 (Direct Compaction) - 针对大块内存



如果申请的是**大块连续内存**（High Order，`costly_order` 为真），单纯回收零散页可能没用，因为内存碎片化太严重了。

1. **直接规整 (`__alloc_pages_direct_compact`)**:
   - 内核尝试移动内存中的页面，把碎片拼凑成大块连续内存。
2. **再次尝试分配**:
   - 如果规整成功，再次尝试分配。

------



### 阶段三：直接回收 (Direct Reclaim) - **真正的“慢速”开始**



如果前面的手段都失效了，说明内存是真的紧缺。此时，内核必须让当前进程**亲自**去干脏活累活。**这是导致系统卡顿、延迟增加的核心代码段。**

1. **检查是否允许阻塞**:
   - `if (!can_direct_reclaim) goto nopage;`
   - 如果当前上下文不允许睡眠（比如在中断处理程序中），那就没办法了，只能放弃，直接跳转到 `nopage` 返回失败。
2. **执行直接回收 (`__alloc_pages_direct_reclaim`)**:
   - **关键点**：当前进程被阻塞，开始扫描 LRU 链表，将脏页写回磁盘，丢弃干净的文件页。
   - **对应指标**：这一步会增加您之前关注的 `pgscan_direct` 计数。
3. **再次尝试分配**:
   - 回收了一部分内存后，再次尝试去 freelist 申请内存。



### 阶段四：循环重试 (Retry Loop)



如果直接回收（Direct Reclaim）之后还没分配到内存，内核面临一个选择：是放弃，还是继续死磕？

1. **判断是否重试 (`should_reclaim_retry`)**:
   - 内核会根据配置（`gfp_mask`）和当前的回收进度（`did_some_progress`）来判断。
   - 如果觉得还有希望（比如刚才回收了一些页，只是还不够），就会跳转回 `retry` 标签，**再次重复阶段三**。
2. **判断是否规整重试 (`should_compact_retry`)**:
   - 如果是大内存申请，检查是否值得再次进行内存规整。



### 阶段五：OOM Killer (核武器)



如果循环重试多次，内存依然无法满足，且没有任何进展，系统进入绝望状态。

1. **触发 OOM (`__alloc_pages_may_oom`)**:
   - 既然无法回收现有内存，那就只能杀掉某些进程来释放内存了。
   - 内核会选择一个“受害者”（通常是内存占用大且非关键的进程），杀掉它，释放它占用的所有物理页。
2. **最后尝试分配**:
   - 杀掉进程后，再次尝试分配。



### 阶段六：最后的倔强 (__GFP_NOFAIL)



如果分配失败，通常就跳转到 `nopage` 返回 NULL 了。但是，有一种特殊情况：

1. **`gfp_mask & __GFP_NOFAIL`**:
   - 如果调用者设置了这个标志，意味着“**我不接受失败，给我死磕到底**”。
   - 代码会进入一个近乎死循环的状态（`goto retry`），不断地压缩、回收、等待，直到分配成功为止。这可能会导致机器看起来彻底“死机”。

```C

```

