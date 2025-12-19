#### 0. 前提

​	经过**root**的安卓设备。

#### 1. 环境可行性验证

**操作：** 通过 ADB 检查设备内核是否暴露了 BTF 信息文件。

```bash
adb shell su -c "ls -l /sys/kernel/btf/vmlinux"
```

**结果：**

- **输出**：`-r--r--r-- 1 root root 5785743 ... /sys/kernel/btf/vmlinux`
- **结论**：文件存在且大小正常。
- **意义**：
  - 说明该 Android 设备的内核在编译时开启了 `CONFIG_DEBUG_INFO_BTF=y`。
  - `/sys/kernel/btf/vmlinux` 包含了该内核中所有数据结构、函数签名和类型定义。
  - **判定**：可以使用现代化 eBPF (CO-RE) 开发模式，无需下载内核源码。

#### 2. 提取 BTF 文件

1. **复制到临时目录**： 直接 `adb pull` 系统目录通常会因为权限不足失败，所以先用 Root 权限将其复制到可读写的临时目录 `/data/local/tmp`。

   ```bash
   adb shell "su -c 'cp /sys/kernel/btf/vmlinux /data/local/tmp/vmlinux.btf'"
   ```

2. **修改权限**： 确保该副本文件是可读的。

   ```bash
   adb shell "su -c 'chmod 644 /data/local/tmp/vmlinux.btf'"
   ```

3. **拉取到 PC**： 将文件从手机传输到电脑当前工作目录。

   ```bash
   adb pull /data/local/tmp/vmlinux.btf .
   ```

4. **清理现场**： 删除手机中的临时文件，保持环境整洁。

   ```bash
   adb shell "rm /data/local/tmp/vmlinux.btf"
   ```

#### 3. 生成 `vmlinux.h` 头文件

在 PC 上使用 `bpftool` 工具进行转换，[bpftool](https://github.com/libbpf/bpftool)需要提前在linux或WSL上安装。

```bash
bpftool btf dump file vmlinux.btf format c > vmlinux.h
```

#### 4. 编写内核态代码

​	 eBPF程序一般分为运行在内核空间的eBPF内核程序和运行在用户空间的eBPF加载程序。eBPF内核程序的代码文件后缀一般为 `.bpf.c` 

​	根据实际需求找AI。注意在prompt中提及，“已生成vmlinux.h, 编写CO-RE eBPF程序。”

​	将内核态代码`.bpf.c`与`vmlinux.h`放在同一目录下。

编译内核态代码

```bash
# 使用 clang 编译，目标架构设为 arm64 (Android)
clang -g -O2 -target bpf -D__TARGET_ARCH_arm64 -c [替换为真实文件名].bpf.c -o [替换为真实文件名].bpf.o
```

#### 6. 编写用户态加载器，编译

​	下面2种方法任选其一。

##### **（1）**使用 Makefile/CMake (C/libbpf) 交叉编译 Android eBPF 程序    *！！！依赖地狱警告*，不推荐

##### （2）使用 Golang (cilium/ebpf) 开发      *！！！强烈推荐*

- 要求已经安装Golang

- 初始化

  ```bash
  mkdir -p [文件夹名称]
  cd [文件夹名称]
  # 把之前 clang 编译出来的 .o 文件拿过来
  cp /path/to/[xxxx].bpf.o .
  # 初始化go项目
  go mod init [项目名称]
  go get github.com/cilium/ebpf
  ```

- 编写用户态加载器

  - 找AI帮忙

- 交叉编译

  ```bash
  CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o [替换为目标ELF文件名] main.go
  ```

- 推送并运行

    ```shell
    adb push [ELF程序名] /data/local/tmp/
    adb shell
    su
    cd /data/local/tmp/
    chmod +x [ELF程序名]
    ./[ELF程序名]
    ```
