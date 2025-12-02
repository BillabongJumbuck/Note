# Towards a Machine Learning-Assisted Kernel with LAKE

### 论文内容简介

​	操作系统依赖于一系列启发式方法进行决策。随着现代计算机硬件与软件的复杂性不断增加，使用机器学习替代这些启发式方法在多个操作系统子模块取得进展。先前的工作仅仅关注于展示ML替代内核模块/策略带来的潜在优点，这篇文章关注将机器学习算法融合进入OS内核的所面临的系统性挑战。

​	在研究一些列ML for OS的工作后，作者发现了以下将机器学习算法融合进入OS内核的系统性挑战：	

1. 使用 GPU/TPU 等专用硬件对于减少 ML 算法的性能影响至关重要。但现有的加速器API，如CUDA，只面向用户空间，没有提供对内核空间的API。
2. 内核ML推理工作可能会与用户空间的工作争夺对加速器设备（GPU）的访问权限，而且与用户空间进程之间的竞争不同，目前还没有明确的机制来管理这种争夺。
3. 在OS内核使用GPU加速ML推理并不总是能带来性能提升，必须考虑在 内核 -> 用户空间 -> GPU 进行数据传输带来的性能开销。 

​	为了应对这些挑战，作者设计并实现了**L**earning-assisted, **A**ccelerated **KE**rnel (**LAKE**)框架

![](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-28%20162554.png)

​	***lakeLib*** 是一个内核模块，它将加速器供应商用户空间库中的 API 作为符号暴露给内核空间。这个模块含有一个与其想要在内核空间中支持的 API 同名的函数。例如，为了在内核空间中支持` cuMemAlloc` 这个 CUDA API，***lakeLib*** 中必须包含一个同名函数。

​	***lakeLib*** 中的每个函数都执行以下三个操作：

1. 将 API 标识符和所有 API 参数序列化成一个命令。

2. 通过某个通信通道传输此命令，以便在用户空间中进行远程执行。

3. 最后，等待响应。

   

​	***lakeD*** 是一个用户空间守护进程（user space daemon），它负责监听来自 ***lakeLib*** 的命令，反序列化这些命令，并执行所请求的 API。该守护进程必须能够访问供应商的库（例如` cudart.so`），才能实现***lakeLib*** 请求的 API。

​	以 `cuMemAlloc` API 为例：

1. 用于该 API 的命令包含一个字段，用于标识要执行哪个 API 及其参数：要分配的字节数以及一个用于存储新分配起始地址的指针。
2. *lakeD* 反序列化该命令以获取这些字段。
3. 它使用供应商的原始库来执行该 API。
4. 最后，它通过接收初始命令的同一通道将结果发回：即返回代码和 API 调用返回的指针。



​	最后，***lakeShm*** 也是一个内核模块，它为 ***lakeLib*** 和使用 **LAKE** 驱动的应用程序提供内存分配服务。通过 ***lakeShm*** 的 API 分配的内存经过优化，可用于内核空间应用程序与用户空间守护进程 ***lakeD*** 之间的数据传输。

​	*lakeShm* 的工作原理是：

1. 向 Linux 内核请求并映射一个大的连续内存区域。
2. 当 ***lakeD*** 启动时，同一区域也会被映射到它的进程空间中。

虽然主机到设备（host-to-device）的传输仍然是必需的，但这使得内核空间模块和 ***lakeD*** 之间能够实现零拷贝（zero-copy）的内存移动。



### Evaluation

#### 0. 环境

- 作者环境

  > All of our evaluation was done on a server with two 16-core Intel Xeon Gold 6226R CPUs, 376 GiB DDR4 RAM, two NVIDIA A100 GPUs and three Samsung 980 Pro 1TB (PCIe 4.0) NVMes. We used Ubuntu 22.04 with our modified Linux kernel based on version 6.0.

- 复现环境

  - Ubuntu 22.04 with modified Linux kernel based on version 6.0.
  - one NVIDIA GeForce RTX 2050 GPU
  - 31744MiB  RAM
  - 13th Gen Intel i5-1335U (12) @ 3.300GHz CPU
  - no NVMes. Three USB flash disk instead, 

#### 1. [LinnOS](https://dl.acm.org/doi/10.5555/3488766.3488776): I/O Latency Prediction

- 环境差异
  - 作者使用三块NVMes设备。复现使用3块U盘代替
  - 由于U盘随机写入速度慢，仅使用3000条数据进行训练（大约训练了40分钟）

- 训练
  - 用对数正态分布随机生成读/写地址
  - 在存储设备上进行读写，收集baseline数据
  - 使用baseline数据进行训练，得到神经网络参数。在用户空间进行训练
  - 输入31个特征，输出结果分为快慢两种

- 测试

  - 在内核空间进行推理。

  - 收集真实的存储设备读写数据，输入神经网络进行推理
  - 根据推理结果，预测IO延迟的快慢，采取不同的措施，以提高吞吐量
  - 测试关注推理花费的时间。横轴为神经网络输入的`batch_size`， 纵轴为推理时间（us)
  - 测试分为CPU和LAKE(GPU)两大组。+1， +2 后缀表示在3层神经网络的基础上增加至4/5层神经网络。

- 下图为复现运行结果

  ![](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/linnos_xover.png)

- 下图为作者运行结果

  <img src="https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/image-20251201201814758.png" alt="image-20251201201814758" style="zoom:150%;" />

#### 2. [Kleio](https://dl.acm.org/doi/10.1145/3307681.3325398): Page Warmth Classification

1. 模型：LSTM-based classifier， 使用tensorflow在用户空间进行推理。通过LAKE框架，在内核调用用户空间实现的`kleioInference` 推理函数。

2. 输入：
   - **没有使用真实的内存页面读取数据。**
   - 使用`int def_inputs[26] = {60, 500, 560, 60, 320, 620, 440, 180, 60, 620, 560, 240, 60, 360, 620, 380, 180, 120, 620, 620, 100, 60, 420, 620, 340, 140};`为固定的输入数据。
   - `def_inputs[26]`按步长大小为6进行分组，得到20个原始样本。
   - 将20个原始样本复制，得到`input_size`为20/80/140/.../1160的LSTM模型输入
   
3. metric:
   - `input_size`为20/80/140/.../1160的情况下，LSTM模型的推理时间（ms)。
   
   - 具体实现为测量`kleioInference` 函数的运行时间，运行10次取平均
   
   - 分为使用CPU与使用LAKE（GPU)两组
   
   - AE_前缀表示该结果是复现运行结果，非AE前缀表示作者运行结果
   
     ![Kleio](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/kleio_ae.png)



#### 3. [MLLB](https://dl.acm.org/doi/10.1145/3409963.3410492): Load Balancing

1. 模型：MLP多层感知机。输入：15个特征；输出：0或1。在内核进行推理。
2. 输入：
   - **没有使用真实的负载均衡数据。**
   - 使用`  int rand_floats_as_int[] = {1036831949, 1045220557, 1050253722, -1110651699};`为固定的输入数据
   - 将`rand_floats_as_int[]`中的**整数按二进制位模式直接解释为浮点数**。得到4个浮点数。
   - 将四个浮点数循环填充到1024 * 15 的矩阵中。
   - 取 1024 * 15 的矩阵的前1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024行，作为MLP模型的推理输入的batch
3. metric
   - 在`batch_size`为1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024的情况下，运行一次MLLB推理的时间（us)。
   - 分为三组：
     - 使用GPU
     - 使用LAKE（GPU)，仅考虑GPU推理的时间
     - 使用LAKE（GPU）并考虑从数据传输到GPU，GPU推理，从GPU获取结果的总时间。
   - AE_前缀表示该结果是复现运行结果，非AE前缀表示作者运行结果
   - **多次运行均出现考虑数据传输的总时间小于仅考虑推理的时间的异常情况！**

![](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/mllb_ae2.png)

#### 4. [KML](https://dl.acm.org/doi/10.1145/3465332.3470875): Filesystem Prefetching

1. 模型：MLP。层次：5 → 15 → 5 → 4。在内核进行推理

2. 输入：

   - **没有使用真实的文件系统数据**
   - 使用`float input[5] = { -0.586797, 5.456822, 5.456966, -0.297318, -1.184651};`作为固定输入。
   - 将`input[5]`复制1/2/4/.../1024次，生成不同`batch_size`的MLP输入。

3. metric

   - 在`batch_size`为1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024的情况下，运行一次KML推理的时间（us)。
   - 分为三组：
     - 使用GPU
     - 使用LAKE（GPU)，仅考虑GPU推理的时间
     - 使用LAKE（GPU）并考虑从数据传输到GPU，GPU推理，从GPU获取结果的总时间。
   - AE_前缀表示该结果是复现运行结果，非AE前缀表示作者运行结果
   - **多次运行均出现考虑数据传输的总时间小于仅考虑推理的时间的异常情况！**
   
   ![KML](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/kml_ae.png)

#### 5. KNN: Malware Detection

1. 模型：KNN。代码只计算距离和找最近邻，没有存储参考点的标签，没有输出分类结果

2. 输入：

   - **没有使用真实的系统调用数据**

   - 使用随机数生成器产生4094个特征数为128的样本点，形成参考集

   - 使用随机数生成器产生4094个特征数为128的样本点，形成查询集

   - 每次选取`batch_size`为`8, 16, 32, 64, 128, 256, 512, 1024`的样本点进行KNN推理

3. metric

   - 在`batch_size`为1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024的情况下，运行一次KNN推理的时间（us)。
   - 分为三组：
     - 使用GPU
     - 使用LAKE（GPU)，仅考虑GPU推理的时间
     - 使用LAKE（GPU）并考虑从数据传输到GPU，GPU推理，从GPU获取结果的总时间。
   - AE_前缀表示该结果是复现运行结果，非AE前缀表示作者运行结果

![KNN](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/knn_ae.png)

#### 6. Contention

​	在内核中使用GPU引入了内核ML任务与用户空间程序对GPU使用的竞争。作者采取的资源分配策略为：当竞争发生时，内核任务让出GPU的使用，保证用户空间程序的计算吞吐量。

实验设计：

1. T0时刻，在内核运行LinnOS，执行IO延迟推理。
   - 以20ms为一轮。在一轮内，以`batch_size=32`的大小为模型输入，记录在20ms内总共完成的batch数量（每完成一次推理加32）
   - 一轮结束后，检查GPU利用率。若使用GPU的进程数量大于1且GPU利用率大于70，退回到CPU进行IO延迟推理。
2. T1时刻，在用户空间启动使用GPU计算文件哈希的程序
3. T2时刻，用户空间程序开始使用GPU
4. T3时刻，用户空间程序终止

![](https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/contention642.png)

- 该运行结果经过cherry pick（大约从30张图像里挑选）。某个参数经过修改，与作者原代码不同。
- 下图为作者运行结果：
  <img src="https://qjm-typora.oss-cn-wuhan-lr.aliyuncs.com/20251201201522987.png" style="zoom: 150%;" />