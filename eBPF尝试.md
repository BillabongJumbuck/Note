### 0. 环境

WSL + Ubuntu + Linux 6.6

```shell
OS: Ubuntu 24.04.3 LTS on Windows 10 x86_64 
Kernel: 6.6.87.2-microsoft-standard-WSL2 
```

> 相关链接：[eBPF on Android](https://codefuturesql.top/post/android_ebpf_basic/)

### 1. 源代码： `mm/page_alloc.c` 4390行（v6.6)

```C
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

- **`get_page_from_freelist`**：这是**快速路径**。它它尝试直接从 **伙伴系统 (Buddy System)** 的空闲链表中找到满足要求的物理页。

- **`if (likely(page))`**：如果成功分配（`likely` 宏提示编译器这是最可能发生的情况），则直接跳转到 `out` 标签返回分配到的页。

- **`__alloc_pages_slowpath`**：这是分配的**慢路径**。只有在快路径失败时才调用。一旦执行这个函数，意味着系统已经处于内存压力之下。

### 2. eBPF探测尝试

#### 1. 对`__alloc_pages_slowpath`探测

```shell
stdin:1:1-30: WARNING: __alloc_pages_slowpath is not traceable (either non-existing, inlined, or marked as "notrace"); attaching to it will likely fail

kprobe:__alloc_pages_slowpath { @slow_path_count = count(); }

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Attaching 1 probe...

cannot attach kprobe, probe entry may not exist

ERROR: Error attaching probe: 'kprobe:__alloc_pages_slowpath'
```

失败原因：`__alloc_pages_slowpath`的函数签名被标记为`inline`

#### 2. 对`get_page_from_freelist`探测

​	可以通过eBPF探测`get_page_from_freelist`的返回值。如果`get_page_from_freelist`返回NULL，表示此次调用快速路径分配失败。

```shell
// 在shell中输入
sudo bpftrace -e '
kretprobe:get_page_from_freelist
{
    if (retval == 0) {
        @slowpath_cnt++;
    }
}
'
// 一段时间后按CTRL C查看结果并退出
```

​	示例：使用stress-ng模拟压力测试

```shell
stress-ng --vm 2 --vm-bytes 95% --vm-keep -t 60s
```

- --vm 2: 启动 2 个虚拟内存压力进程。

- --vm-bytes 95%: 试图占用 95% 的物理内存。

- --vm-keep: 保持内存占用，不释放，强迫内核在剩余的 5% 空间里挣扎，从而频繁触发慢速路径。

​	输出：

```shell
$ sudo bpftrace -e '
    kretprobe:get_page_from_freelist
    {
        if (retval == 0) {
            @slowpath_cnt++;
        }
    }
    '
[sudo] password for: 
Attaching 1 probe...
^C

@slowpath_cnt: 89551
```

eBPF程序运行期间`get_page_from_freelist`共返回了89551次`NULL`

#### 3.  对`get_page_from_freelist`探测的缺陷

​	在`__alloc_pages_slowpath`中会多次调用`get_page_from_freelist`，实际进行慢速分配的次数会少于`get_page_from_freelist`返回`NULL`的值。

`__alloc_pages_slowpath`流程简介（结合ChatGPT)：

1. 重新计算`alloc_flags`，降低内存使用的标准;唤醒后台的 kswapd 守护进程。让它在后台异步地回收内存，希望它能赶在当前进程彻底没内存之前释放出一些空间。再次调用`get_page_from_freelist`。
2. 如果申请的是大块连续内存，单纯回收零散页可能没用，内核尝试移动内存中的页面，把碎片拼凑成大块连续内存。规整成功，再次调用。`get_page_from_freelist`。
3. 尝试阻塞进程，开始扫描 LRU 链表，将脏页写回磁盘，丢弃干净的文件页。回收了一部分内存后，再次调用`get_page_from_freelist。`
4. 内核会根据配置（`gfp_mask`）和当前的回收进度（`did_some_progress`）来判断是否重试。如果重试， 返回步骤3。
5. 触发 OOM ，再次尝试分配。
6. 分配页面失败。



