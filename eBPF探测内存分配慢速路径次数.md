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

#### 4. 另一种方法

​	注意到，在内核中，对`get_page_from_freelist`的调用分布在：

``` 
mm/page_alloc.c


__alloc_pages()
    |-- get_page_from_freelist()
    |-- __alloc_pages_slowpath
        |-- get_page_from_freelist()
        |-- get_page_from_freelist()
        |-- __alloc_pages_cpuset_fallback()
            |-- get_page_from_freelist()
            |-- get_page_from_freelist()
    	|-- __alloc_pages_may_oom()
            |-- get_page_from_freelist()
            |-- __alloc_pages_cpuset_fallback()
                |-- get_page_from_freelist()
                |-- get_page_from_freelist()
		|-- __alloc_pages_direct_compact()
            |-- get_page_from_freelist()
        |-- __alloc_pages_direct_compact()
            |-- get_page_from_freelist()
        |-- __alloc_pages_direct_reclaim()
            |-- get_page_from_freelist()
```

在 `__alloc_pages` 的源码逻辑中：

1. 进入 `__alloc_pages`。

2. **第一次**调用 `get_page_from_freelist`。

3. 如果这次返回 `NULL` (0)，**定义上**就是进入了慢速路径。

4. 之后无论发生什么（重试、循环），都不再是“第一次”了。

只需要用 BPF 脚本实现这个逻辑：**“在 `__alloc_pages` 内部，拦截第一次 `get_page_from_freelist` 的返回结果。”**



#### 5. 数据

```shell
OP5D2BL1:/data/local/tmp # ./slowpath_android
eBPF Running on Android (Kprobe mode)... (Ctrl+C to stop)
TIME       COUNT      USED(MB)   USAGE(%)
17:49:01   0          4665       30.69%
17:49:02   0          4665       30.69%
17:49:03   0          4665       30.69%
17:49:04   0          4655       30.62%
17:49:05   0          4685       30.82%
17:49:06   0          4691       30.86%
17:49:07   0          4709       30.97%
17:49:08   0          4937       32.47%
17:49:09   0          4702       30.93%
17:49:10   0          4693       30.87%
17:49:11   0          4716       31.02%
17:49:12   0          4860       31.97%
17:49:13   0          4964       32.65%
17:49:14   0          4972       32.71%
17:49:15   0          4914       32.33%
17:49:16   0          4955       32.59%
17:49:17   0          4986       32.80%
17:49:18   0          5377       35.37%
17:49:19   0          5287       34.77%
17:49:20   0          5494       36.14%
17:49:21   0          5515       36.27%
17:49:22   0          5577       36.68%
17:49:23   0          5599       36.83%
17:49:24   0          5609       36.89%
17:49:25   0          5685       37.40%
17:49:26   0          5675       37.33%
17:49:27   0          5570       36.64%
17:49:28   0          5530       36.38%
17:49:29   0          5594       36.79%
17:49:30   0          5712       37.57%
17:49:31   0          5649       37.16%
17:49:32   78         6221       40.92%
17:49:33   81         6758       44.45%
17:49:34   82         6703       44.09%
17:49:35   82         6251       41.11%
17:49:36   83         6252       41.12%
17:49:37   83         6304       41.47%
17:49:38   91         6723       44.22%
17:49:39   127        6775       44.56%
17:49:40   129        6580       43.28%
17:49:41   129        6485       42.66%
17:49:42   129        6439       42.35%
17:49:43   129        6458       42.48%
17:49:44   129        6449       42.42%
17:49:45   129        6459       42.49%
17:49:46   129        6470       42.56%
17:49:47   129        6491       42.69%
17:49:48   129        6509       42.81%
17:49:49   129        6723       44.22%
17:49:50   129        6661       43.81%
17:49:51   129        6554       43.11%
17:49:52   129        6562       43.16%
17:49:53   129        6543       43.04%
17:49:54   129        6520       42.88%
17:49:55   129        6653       43.76%
17:49:56   129        6630       43.61%
17:49:57   129        6735       44.30%
17:49:58   129        6753       44.42%
17:49:59   129        6791       44.67%
17:50:00   129        6806       44.77%
17:50:01   129        6811       44.80%
17:50:02   129        6824       44.89%
17:50:03   129        6823       44.88%
17:50:04   129        6925       45.55%
17:50:05   129        6997       46.02%
17:50:06   129        6994       46.01%
17:50:07   129        7016       46.15%
17:50:08   129        7090       46.64%
17:50:09   129        7088       46.62%
17:50:10   129        7082       46.58%
17:50:11   129        7114       46.80%
17:50:12   129        7117       46.82%
17:50:13   129        7158       47.08%
17:50:14   129        7051       46.38%
17:50:15   132        7194       47.32%
17:50:16   212        7659       50.38%
17:50:17   212        7436       48.91%
17:50:18   212        7515       49.43%
17:50:19   212        7055       46.40%
17:50:20   212        7177       47.21%
17:50:21   212        7242       47.64%
17:50:22   212        7262       47.77%
17:50:23   212        7246       47.66%
17:50:24   212        7392       48.62%
17:50:25   212        7379       48.54%
17:50:26   212        7379       48.54%
17:50:27   212        7287       47.93%
17:50:28   212        7288       47.94%
17:50:29   212        7286       47.93%
17:50:30   212        7312       48.09%
17:50:31   212        7329       48.20%
17:50:32   213        7522       49.48%
17:50:33   213        7627       50.17%
17:50:34   214        7657       50.36%
17:50:35   214        7547       49.64%
17:50:36   214        7578       49.84%
17:50:37   219        7813       51.39%
17:50:38   219        7772       51.12%
17:50:39   219        7766       51.08%
17:50:40   220        7955       52.32%
17:50:41   221        7917       52.08%
17:50:42   221        7952       52.31%
17:50:43   221        7902       51.97%
17:50:44   221        7901       51.97%
17:50:45   221        7853       51.66%
17:50:46   221        7860       51.70%
17:50:47   221        7984       52.52%
17:50:48   221        7966       52.40%
17:50:49   221        7964       52.38%
17:50:50   221        8032       52.83%
17:50:51   222        8054       52.98%
17:50:52   222        8080       53.14%
17:50:53   224        8101       53.29%
17:50:54   238        8339       54.85%
17:50:55   238        8336       54.83%
17:50:56   238        8205       53.97%
17:50:57   241        8288       54.51%
17:50:58   248        8203       53.96%
17:50:59   248        8211       54.01%
17:51:00   248        8488       55.83%
17:51:01   248        8175       53.77%
17:51:02   248        7967       52.40%
17:51:03   248        7970       52.43%
17:51:04   248        7761       51.05%
17:51:05   248        7782       51.19%
17:51:06   248        7907       52.01%
17:51:07   248        7738       50.90%
17:51:08   248        7759       51.03%
17:51:09   248        7761       51.05%
17:51:10   248        7699       50.64%
17:51:11   248        7682       50.53%
17:51:12   248        7791       51.25%
17:51:13   248        7688       50.57%
17:51:14   248        7689       50.57%
17:51:15   248        7743       50.93%
17:51:16   248        7692       50.59%
17:51:17   248        7850       51.63%
17:51:18   248        8022       52.77%
17:51:19   248        8042       52.90%
17:51:20   248        8189       53.87%
```

