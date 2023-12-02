---
Title: Rust for Linux驱动模块开发报告-lcx
Date: 2023-11
Tags:
    - Author: lcx
    - Reports Repo: https://github.com/451846939/TimeMachine/tree/master/src/os/rust-for-linux
---

# rust for linux

## 1. 背景 

Rust for Linux 这个项目的目的就是为了将 Rust 引入 Linux，让 Rust 成为 C 语言之后的第二语言。但它最初的目的是：实验性地支持Rust来写内核驱动。

以往，Linux 内核驱动的编写相对于应用其实是比较复杂的，具体复杂性主要表现在以下两个方面：

- 编写设备驱动必须了解Linux 内核基础概念、工作机制、硬件原理等知识
- 设备驱动中涉及内存和多线程并发时容易出现 Bug，linux驱动跟linux内核工作在同一层次，一旦发生问题，很容易造成内核的整体崩溃

引入 Rust 从理想层面来看，一方面在代码抽象和跨平台方面比 C 更有效，另一方面代码质量会更高，有效减少内存和多线程并发类 Bug 。但具体实践如何，是否真如理想中有效，这就需要后续的实验。

Rust for Linux 就是为了帮助实现这一目标，为 Linux 提供了 Rust 相关的基础设施和方便编写 Linux 驱动的安全抽象。

### [Rust for Linux 第四次补丁提审](https://rustmagazine.github.io/rust_magazine_2022/Q1/contribute/rust-for-linux-clk.html#rust-for-linux-第四次补丁提审)

补丁的具体细节可以在Linux 邮件列表中找到：

- RFC: https://lore.kernel.org/lkml/20210414184604.23473-1-ojeda@kernel.org/
- v1: https://lore.kernel.org/lkml/20210704202756.29107-1-ojeda@kernel.org/
- v2: https://lore.kernel.org/lkml/20211206140313.5653-1-ojeda@kernel.org/
- v3: https://lore.kernel.org/lkml/20220117053349.6804-1-ojeda@kernel.org/

第二次补丁改进摘要可参考：[Rust for Linux 源码导读 | Ref 引用计数容器](https://rustmagazine.github.io/rust_magazine_2021/chapter_12/ref.html)。

**第三次补丁改进摘要：**

- 对 Rust 的支持有一些改进：
  - 升级到 Rust 1.58.0
  - 增加了自动检测来判断是否有合适的 Rust 可用工具链（`CONFIG_RUST_IS_AVAILABLE`，用于替换`HAS_RUST`）
  - 移除`!COMPILE_TEST`
  - 其他构建系统的改进
  - 文档改进
  - 需要的不稳定功能之一，`-Zsymbol-mangling-version=v0`，在 1.59.0 中变得稳定。另一个，“maybe_uninit_extra”，将在 1.60.0 中。
- 对抽象和示例驱动程序的一些改进：
  - 加了将在总线中使用的“IdArray”和“IdTable”，以允许驱动程序指定在编译时保证为零终止（zero-terminated）的设备 ID 表。
  - 更新了 `amba` 以使用新的常量设备 ID 表支持。
  - 初始通用时钟框架抽象。
  - 平台驱动程序现在通过实现特质（trait）来定义。包括用于简化平台驱动程序注册的新宏和新示例/模板。
  - `dev_*` 打印宏。
  - `IoMem<T>` 的`{read,write}*_relaxed` 方法。
  - 通过删除 `FileOpener` 来简化文件操作。
  - 在驱动程序注册的参数中添加了“ThisModule”。
  - 添加用 Rust 编写的树外 Linux 内核模块的基本模板： [https ://github.com/Rust-for-Linux/rust-out-of-tree-module](https ://github.com/Rust-for-Linux/rust-out-of-tree-module)

**第四次补丁改进摘要：**

- 基础设施更新：

  - 整合 CI : 英特尔 0DAY/LKP 内核测试机器人 / kernelCI/ GitHub CI
  - 内核模块不需要写 crate 属性，`#![no_std]` 和 `#![feature(...)]` 不再存在，删除样板。
  - 添加了单目标支持，包括`.o`、`.s`、`.ll`和`.i`（即宏扩展，类似于 C 预处理源）。
  - 对`helpers.c`文件的解释和对helpers的许可和导出的许可。
  - 文档Logo现在基于矢量 (SVG)。此外，已经为上游提出了 Tux 的矢量版本，并且用于改进自定义Logo支持的 RFC 已提交至上游 Rust。
  - 添加了关于注释 (`//`) 和代码文档的编码指南(`///`)。
  - `is_rust_module.sh` 返工。
  - 现在跳过了叶子模块的`.rmeta`的生成。
  - 其他清理、修复和改进。

- 抽象和驱动更新:

  - 增加了对静态（全局共享变量）同步原语的支持。`CONFIG_CONSTRUCTORS`被用于实现。

  - 通过使用标记类型简化了锁防护，即`Guard`和`GuardMut`统一为一个参数化的类型。如果标记是 `WriteLock`，那么 `Guard`就会实现`DerefMut`（以前只由`GuardMut`实现）。

  - 可选参数添加到杂项设备（misc device）的注册中。遵循构建者模式，例如:

    ```rust
    miscdev::Options::new()
        .mode(0o600)
        .minor(10)
        .parent(parent)
        .register(reg, c_str!("sample"), ())
    ```

  - 增加了 "RwSemaphore "的抽象，该抽象包裹了C端`struct rw_semaphore`。

  - 新的`mm`模块和VMA抽象（包装C端`struct vm_area_struct`）用于`mmap`。

  - GPIO PL061现在使用最近增加的`dev_*!` Rust宏。

  - 支持`！CONFIG_PRINTK`情况。

  - 其他清理、修复和改进。

rust驱动和内核的关系正如下图：

![image-20231117155649678](https://github.com/451846939/TimeMachine/blob/master/src/os/rust-for-linux/rust-for-linux.assets/image-20231117155649678-0215557.png)

## 2. <a name="2">编译</a>

> https://rust-for-linux.com
>
> https://www.kernel.org/doc/html/next/rust/quick-start.html
>
> https://github.com/Rust-for-Linux/linux

首先 

```shell
git clone https://github.com/Rust-for-Linux/linux.git
cd linux
make LLVM=1 rustavailable
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
```

根据提示[官方文档](https://www.kernel.org/doc/html/next/rust/quick-start.html) 配置好你的rust环境最重要的是要注意llvm要使用16以上的版本

这里给出llvm如何配置apt 以及如何安装clang+llvm等开发环境的文档地址： https://apt.llvm.org

基本需要的：

```shell
sudo apt-get -y install \
  binutils build-essential libtool texinfo \
  gzip zip unzip patchutils curl git \
  make cmake ninja-build automake bison flex gperf \
  grep sed gawk bc \
  zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev libelf-dev libssl-dev
```

```shell
apt-get install clang-format clang-tidy clang-tools clang clangd libc++-dev libc++1 libc++abi-dev libc++abi1 libclang-dev libclang1 liblldb-dev libllvm-ocaml-dev libomp-dev libomp5 lld lldb llvm-dev llvm-runtime llvm python3-clang
```

llvm和clang请注意版本，如果安装的clang16可能运行文件名字为clang-16 可以使用

```she
ln -s /usr/bin/clang-16 /usr/bin/clang
```

然后

```shell
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig
#set the following config to yes
General setup
        ---> [*] Rust support
Kernel hacking
    ---> Sample kernel code
        ---> [*] Rust samples
cd build
make ARCH=arm64 LLVM=1 -j8
```

menuconfig 会进入一个菜单选择，记得打开`Rust support` 和`Rust samples` 

![image-20231107214913238](https://github.com/451846939/TimeMachine/blob/master/src/os/rust-for-linux/rust-for-linux.assets/image-20231107214913238.png)

## 3. 自定义内核驱动模块

我们可以先前往linux的`samples/rust/` 下可以看到rust的驱动例子，那我们可以仿造来写一个hello_world

创建`rust_helloworld.rs`

```rust
use kernel::prelude::*;
      
module! {
  type: RustHelloWorld,
  name: "rust_helloworld",
  author: "whocare",
  description: "hello world module in rust",
  license: "GPL",
}
      
struct RustHelloWorld {}
      
impl kernel::Module for RustHelloWorld {
  fn init(_module: &'static ThisModule) -> Result<Self> {
      pr_info!("Hello World from Rust module");
      Ok(RustHelloWorld {})
  }
}
```

在rust目录下的Makefile加入`obj-$(CONFIG_SAMPLE_RUST_HELLOWORLD)        += rust_helloworld.o`

```rust
# SPDX-License-Identifier: GPL-2.0

obj-$(CONFIG_SAMPLE_RUST_MINIMAL)		+= rust_minimal.o
obj-$(CONFIG_SAMPLE_RUST_PRINT)			+= rust_print.o
// +++++++++++++++ add here
obj-$(CONFIG_SAMPLE_RUST_HELLOWORLD)        += rust_helloworld.o
// +++++++++++++++ 

subdir-$(CONFIG_SAMPLE_RUST_HOSTPROGS)		+= hostprogs
```

在Kconfig中加入

```Kconfig
# SPDX-License-Identifier: GPL-2.0

menuconfig SAMPLES_RUST
	bool "Rust samples"
	depends on RUST
	help
	  You can build sample Rust kernel code here.

	  If unsure, say N.

if SAMPLES_RUST

config SAMPLE_RUST_MINIMAL
	tristate "Minimal"
	help
	  This option builds the Rust minimal module sample.

	  To compile this as a module, choose M here:
	  the module will be called rust_minimal.

	  If unsure, say N.

config SAMPLE_RUST_PRINT
	tristate "Printing macros"
	help
	  This option builds the Rust printing macros sample.

	  To compile this as a module, choose M here:
	  the module will be called rust_print.

	  If unsure, say N.
// +++++++++++++++ add here
config SAMPLE_RUST_HELLOWORLD
	tristate "Print Helloworld in Rust"
	help
		This option builds the Rust HelloWorld module sample.
		
		To compile this as a module, choose M here:
		the module will be called rust_helloworld.
		
		If unsure, say N.
// +++++++++++++++
config SAMPLE_RUST_HOSTPROGS
	bool "Host programs"
	help
	  This option builds the Rust host program samples.

	  If unsure, say N.

endif # SAMPLES_RUST
```

回到linux目录

```shell
make ARCH=arm64 LLVM=1 O=build menuconfig
```

这时候我们可以去

```test
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <*>Print Helloworld in Rust (NEW)
```

打开后

```shell
cd build
make ARCH=arm64 LLVM=1 -j8
```



## 4. qemu启动写的驱动

### 1. 方法1

之后需要用qemu来进行内核的运行并且要制作一个initramfs

我们需要用到busybox

这里我们先下载一个由于写的时候busybox最新版是`1.36.1` 所以下载`1.36.1`

```shell
cd ~
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2

tar -xf busybox-1.36.1.tar.bz2
cd busybox-1.36.1

make menuconfig ARCH=arm64
# 修改配置，选中如下项目，静态编译
# Settings -> Build Options -> [*] Build static binary (no share libs)

make -j8

cp ~/busybox-1.36.1/busybox rust-for-linux/linux/
```

完整的流程[busybox制作initramfs](./使用busybox制作内存文件系统initramfs.md)  在这里我们到这一步暂时就够了 当然你也可以自己从[0构建一个](https://docs.kernel.org/admin-guide/initrd.html)



这里我们使用rust-for-linux 的rust 分支 .github中workflows来操作

在linux源码目录创建一个qemu-ini.sh

```shell
#!/bin/sh

busybox insmod rust_print.ko
busybox  rmmod rust_print.ko

busybox insmod rust_helloworld.ko
busybox  rmmod rust_helloworld.ko

busybox insmod rust_minimal.ko
busybox  rmmod rust_minimal.ko

busybox reboot -f
```

再创建一个qemu-initramfs.desc

```shell
dir     /bin                                          0755 0 0
dir     /sys                                          0755 0 0
dir     /dev                                          0755 0 0
file    /bin/busybox  busybox                         0755 0 0
slink   /bin/sh       /bin/busybox                    0755 0 0
file    /init         ./qemu-init.sh								  0755 0 0

file    /rust_minimal.ko            build/samples/rust/rust_minimal.ko          0755 0 0
file    /rust_print.ko              build/samples/rust/rust_print.ko            0755 0 0
file    /rust_helloworld.ko          build/samples/rust/rust_helloworld.ko      0755 0 0
```

运行

```shell
build/usr/gen_init_cpio ./qemu-initramfs.desc > qemu-initramfs.img
```

```shell

qemu-system-aarch64 \
  -kernel build/arch/arm64/boot/Image.gz \
  -initrd qemu-initramfs.img \
  -M virt \
  -cpu cortex-a72 \
  -smp 2 \
  -nographic \
  -vga none \
  -no-reboot \
  -append 'root=/dev/sda' \
  | sed 's:\r$::'
```

可以看到如下输出：

```text
....................
[    0.381944]   No soundcards found.
[    0.383772] uart-pl011 9000000.pl011: no DMA platform data
[    0.442688] Freeing unused kernel memory: 2112K
[    0.443427] Run /init as init process
[    0.481102] rust_print: Rust printing macros sample (init)
[    0.481270] rust_print: Emergency message (level 0) without args
[    0.481333] rust_print: Alert message (level 1) without args
[    0.481391] rust_print: Critical message (level 2) without args
[    0.481455] rust_print: Error message (level 3) without args
[    0.481516] rust_print: Warning message (level 4) without args
[    0.481577] rust_print: Notice message (level 5) without args
[    0.481674] rust_print: Info message (level 6) without args
[    0.481742] rust_print: A line that is continued without args
[    0.481837] rust_print: Emergency message (level 0) with args
[    0.481896] rust_print: Alert message (level 1) with args
[    0.481957] rust_print: Critical message (level 2) with args
[    0.482021] rust_print: Error message (level 3) with args
[    0.482102] rust_print: Warning message (level 4) with args
[    0.482173] rust_print: Notice message (level 5) with args
[    0.482234] rust_print: Info message (level 6) with args
[    0.482292] rust_print: A line that is continued with args
[    0.482453] rust_print: 1
[    0.482596] rust_print: "hello, world"
[    0.482884] rust_print: [../samples/rust/rust_print.rs:34] c = "hello, world"
[    0.483099] rust_print: "hello, world"
[    0.489795] rust_print: Rust printing macros sample (exit)
[    0.510844] rust_helloworld: Hello World from Rust module
[    0.546105] rust_minimal: Rust minimal sample (init)
[    0.546284] rust_minimal: Am I built-in? false
[    0.550119] rust_minimal: My numbers are [72, 108, 200]
[    0.550376] rust_minimal: Rust minimal sample (exit)
[    0.576403] Flash device refused suspend due to active operation (state 20)
[    0.576539] Flash device refused suspend due to active operation (state 20)
[    0.576833] reboot: Restarting system
```

可以看到我们自己写的 Hello World from Rust module已经打印出来了







### <a name="4.2">2. 方法2</a>

也可以直接下载一个[debian](https://people.debian.org/~gio/dqib/)的

debian的镜像解压后可以直接修改 -kernel 为rust-for-linux/linux/build/arch/arm64/boot/Image.gz 以及image.qcow2

```shell
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel Image -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"
```

由于image.qcow2是虚拟镜像格式所以挂载需要一点特殊的技巧

```shell
modprobe nbd max_part=12
qemu-nbd --connect=/dev/nbd0 image.qcow2
mkdir /mnt/test
mount /dev/nbd0p1 /mnt/test/
cp rust-for-linux/linux/build/samples/rust/rust_helloworld.ko /mnt/test/rust_helloworld.ko
umount /mnt/test
qemu-nbd --disconnect /dev/nbd0
modprobe -r nbd
```

再次运行

```shell
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel rust-for-linux/linux/build/arch/arm64/boot/Image.gz -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"
```

之后会进入debian 输入账号密码 root root

直接

```shell
dmesg -C
insmod /rust_helloworld.ko
#[   57.144725] rust_helloworld: Hello World from Rust module
dmesg
rmmod rust_helloworld
```



## 5.和内核函数互相调用

由于rust-for-linux还在发展中有很多内核函数是没有被封装的，利用的是bindgen来自动生成对C（和一些C++）库的Rust FFI绑定

所以我们必须要知道rust-for-linux如何调用c

> https://rust-lang.github.io/rust-bindgen/introduction.html
>
> 这是官方文档
>
> 对于linux是如何使用的可以看
>
> https://github.com/d0u9/Linux-Device-Driver-Rust/blob/master/00_Introduction_to_Rust_Module_in_Linux/03-Rust_build_processes_in_kernel.md

linux 对于bindgen的使用是命令行的方式使用，并且放在了Makeflie中处理流程如下

[![Dependency Graph](https://github.com/451846939/TimeMachine/blob/master/src/os/rust-for-linux/rust-for-linux.assets/dependency_graph.png)](https://github.com/d0u9/Linux-Device-Driver-Rust/blob/master/00_Introduction_to_Rust_Module_in_Linux/dependency_graph.png)

所以我们怎么使用呢在` /rust/kernel/bindings_helper.h`中添加内核头文件

例如:

```c
#include <linux/pci.h>
```

之后编写你的c函数比如pci中`pci_set_drvdata` 生成`rust_helper_pci_set_drvdata` rust函数

添加在`/rust/kernel/helpers.c`中

```c
#include <linux/pci.h>
void rust_helper_pci_set_drvdata(struct pci_dev *pdev, void *data)
{
    pci_set_drvdata(pdev, data);
}
EXPORT_SYMBOL_GPL(rust_helper_pci_set_drvdata);
```

在编译内核的时候会在`/linux/build/rust/bindings`中产生下面2个文件

- bindings_generated.rs
- bindings_helpers_generated.rs

![image-20231117154615901](https://github.com/451846939/TimeMachine/blob/master/src/os/rust-for-linux/rust-for-linux.assets/image-20231117154615901-0207180-0215569.png)

至此可以使用`use kernel::bindings` 来使用`rust_helper_pci_set_drvdata`



## 6. 模块化驱动开发e1000网卡驱动

> 我们可以在linux目录之外编写我们的自定义驱动模块，具体参考如下链接
>
> https://github.com/Rust-for-Linux/rust-out-of-tree-module

### 0.基础知识

>  关于e1000在MIT 6.S081中有相关的介绍
>
> https://pdos.csail.mit.edu/6.S081/2020/labs/net.html
>
> https://pdos.csail.mit.edu/6.S081/2020/readings/8254x_GBe_SDM.pdf



首先我们要了解什么是网卡，网卡和操作系统的交互可以看[参考](./网卡框架.pdf)

我们需要了解[e1000](./8254x_GBe_SDM.pdf) 的相关知识，重点看以下：

- 第2部分是必不可少的，并提供了整个设备的概述。
- 第3.2部分概述了数据包接收的过程。
- 第3.3部分概述了数据包传输，以及第3.4部分。
- 第13部分提供了E1000使用的寄存器的概述。
- 第14部分可能有助于理解我们提供的初始化代码。



### 1.环境准备

#### 仓库准备

这里我们使用别人已经写过的一些方法仓库不然从0开始写驱动需要进行大量的工作我这里直接fork了一份支持生成外部rust-analyzer

e1000使用清华os的训练营仓库，只需要填写checkpoint即可，可以省去一些基础工作，不过建议仔细看

> linux: https://github.com/451846939/rust-for-linux-e1000/tree/rust-e1000
>
> e1000: https://github.com/451846939/e1000-driver/

ps：linux内核其实有一份c的e1000驱动所以我们还需要关闭他们然后重新编译内核,clang和llvm推荐使用14

编译内核的方法和[2.编译](#2)一样只是这里的环境需要修改按照本身的版本进行

如果出现了bindgen 0.56.0下载不下来

可以去 https://github.com/rust-lang/rust-bindgen.git 下载然后切换分支使用cargo install 以及可以把bindgen-cli的cli去掉如下：

```shell
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
```





#### 开启代码提示

linux目录中使用

```shell
make LLVM=1 O=build rust-analyzer
```

目录之外如果我们需要开启rust-analyzer提示需要在代码的src平级目录使用

```shell
make LLVM=1 -C /mnt/rust-for-linux/linux/build M=$PWD rust-analyzer
```

/mnt/rust-for-linux/linux 替换成你自己的linux目录



在vscode `.vscode/settings.json`中添加如下

```json
    "rust-analyzer.linkedProjects": [
        "${workspaceFolder}/e1000-driver/rust-project.json",
        "${workspaceFolder}/linux/build/rust-project.json"
    ],
```



### 2.完成checkpoint




1. 首先在网卡驱动初始化的时候我们需要分配tx_ring和rx_ring的内存空间并返回dma虚拟地址和物理地址

```rust
let (tx_ring_vaddr, tx_ring_dma) = kfn.dma_alloc_coherent(alloc_tx_ring_pages);
let (rx_ring_vaddr, rx_ring_dma) = kfn.dma_alloc_coherent(alloc_rx_ring_pages);
```

对于这里我们一定要理解ring 

![image-20231117162131653](https://github.com/451846939/TimeMachine/blob/master/src/os/rust-for-linux/rust-for-linux.assets/image-20231117162131653-0215580.png)

2. 接着我们需要分配tx_buffer和rx_buffer的内存空间 并返回dma虚拟地址和物理地址 

```rust
let (mut tx_mbufs_vaddr, mut tx_mbufs_dma) =kfn.dma_alloc_coherent(alloc_tx_buffer_pages);
let (mut rx_mbufs_vaddr, mut rx_mbufs_dma) =kfn.dma_alloc_coherent(alloc_rx_buffer_pages);
```



根据MIT 6.S081的HINT得知

`e1000_transmit`的流程

1. 首先，通过获取E1000_RDT控制寄存器并加一模RX_RING_SIZE，询问E1000下一个等待接收的数据包（如果有）的环索引。
2. 然后，通过在描述符的状态部分检查E1000_RXD_STAT_DD位来检查是否有新的数据包可用。如果没有，停止。
3. 否则，将mbuf的`m->len`更新为描述符中报告的长度。使用`net_rx()`将mbuf传递给网络栈。
4. 然后，使用`mbufalloc()`分配一个新的mbuf以替换刚刚传递给`net_rx()`的mbuf。将其数据指针（`m->head`）编程到描述符中。将描述符的状态位清零。
5. 最后，更新E1000_RDT寄存器为最后处理的环描述符的索引。



`e1000_recv`流程：

1. 通过读取E1000_TDT控制寄存器，询问E1000它期望下一个数据包的TX环索引。
2. 然后检查环是否溢出。如果在由E1000_TDT索引的描述符中未设置E1000_TXD_STAT_DD，说明E1000尚未完成相应的先前传输请求，因此返回错误。
3. 否则，使用`mbuffree()`释放从该描述符传输的上一个mbuf（如果有的话）。
4. 接着，填充描述符。`m->head`指向内存中包的内容，`m->len`是包的长度。设置必要的cmd标志（查看E1000手册中的第3.3节），并储存mbuf的指针以供稍后释放。
5. 最后，通过将E1000_TDT模TX_RING_SIZE加一来更新环位置。
6. 如果`e1000_transmit()`成功将mbuf添加到环中，返回0。如果失败（例如，没有可用的描述符传输mbuf），返回-1，以便调用者知道释放mbuf。



这个mbuffer其实就是ring 里做了一个备份

3. 给寄存器设置ring

```rust
// set tx descriptor base address and tx ring length
self.regs[E1000_TDBAL].write(self.tx_ring_dma as u32);
self.regs[E1000_TDLEN].write((self.tx_ring.len() * size_of::<TxDesc>()) as u32);

// set rx descriptor base address and rx ring length
self.regs[E1000_RDBAL].write(self.rx_ring_dma as u32);
self.regs[E1000_RDLEN].write((self.rx_ring.len() * size_of::<RxDesc>()) as u32);
```



4. 设置中断处理

```rust

// Enable interrupts
// step1 set up irq_data
// step2 request_irq
// step3 set up irq_handler
let irq_data = Box::try_new(IrqData {
    dev_e1000: data.dev_e1000.clone(),
    res: data.res.clone(),
    napi: data.napi.clone(),
})?;
let irq_regist = request_irq(data.irq, irq_data)?;
data.irq_handler
    .store(Box::into_raw(Box::try_new(irq_regist)?), Ordering::Relaxed);
```



5. 收包中断处理

```rust
fn handle_rx_irq(dev: &net::Device, napi: &Napi, data: &NetData) {
        // Exercise4 Checkpoint 1
        let mut packets = 0;
        let mut bytes = 0;
        let recv_vec: Option<Vec<Vec<u8>>> = {
            let mut dev_e1k = data.dev_e1000.lock();
            dev_e1k.as_mut().unwrap().e1000_recv()
        };
        if let Some(vec) = recv_vec {
            packets = vec.len();
            vec.into_iter().for_each(|packet| {
                let mut len = packet.len();
                let skb = dev.alloc_skb_ip_align(RXBUFFER).unwrap();
                let skb_buf =
                    unsafe { from_raw_parts_mut(skb.head_data().as_ptr() as *mut u8, len) 						};
                skb_buf.copy_from_slice(&packet);

                skb.put(len as u32);
                let protocol = skb.eth_type_trans(dev);
                skb.protocol_set(protocol);

                napi.gro_receive(&skb);

                bytes += len;
            });
            pr_info!("handle_rx_irq {} packets,{} bytes\n", packets, bytes);
        } else {
            pr_info!("handle_rx_irq no packets\n");
        }

        data.stats
            .rx_bytes
            .fetch_add(bytes as u64, Ordering::Relaxed);
        data.stats
            .rx_packets
            .fetch_add(packets as u64, Ordering::Relaxed);
}
```

首先调用收包函数获取收到的包之后，copy到linux中的核心数据结构skb中设置协议栈后把skb发送到linux网络协议栈，最后更新 `data.stats` 中的 `rx_bytes` 和 `rx_packets` 统计信息



6. 数据发送

```rust
  /// Corresponds to `ndo_start_xmit` in `struct net_device_ops`.
  fn start_xmit(
      skb: &SkBuff,
      dev: &Device,
      data: <Self::Data as PointerWrapper>::Borrowed<'_>,
  ) -> NetdevTx {
      pr_info!("start xmit\n");
      // Exercise4 Checkpoint 2
      skb.put_padto(bindings::ETH_ZLEN);
      let skb_data = skb.len() - skb.data_len();
      let skb_data = skb.head_data();
      dev.sent_queue(skb.len());

      let mut dev_e1k = data.dev_e1000.lock_irqdisable();
      let len = dev_e1k.as_mut().unwrap().e1000_transmit(skb_data);
      drop(dev_e1k);
      if len < 0 {
          pr_warn!("skb packet:{}", len);
          return net::NetdevTx::Busy;
      }
      let bytes = skb.len();
      let packets = 1;

      skb.napi_consume(64);
      data.stats
          .tx_bytes
          .fetch_add(bytes as u64, Ordering::Relaxed);

      data.stats.tx_packets.fetch_add(packets, Ordering::Relaxed);
      dev.completed_queue(packets as u32, bytes as u32);

      return net::NetdevTx::Ok;
  }
```

首先我们要设置以太网帧的最小长度，在设备中记录已发送的数据包长度，调用 `e1000_transmit` 方法，把数据包发送到 E1000 设备， 重新启用中断，如果发送数据包的时候没有成功，可能会存在数据正在接收的情况，所以返回Busy。之后在dev设备中记录已完成的队列和统计信息。



### 3.验证

> linux内核其实有一份c的e1000驱动所以我们还需要关闭它们然后重新编译内核
>
> ```shell
> make ARCH=arm64 LLVM=1 O=build menuconfig
> ```
>
> 使用/ 搜索e1000按数字键 就可以直接到达对应的地方，和 e1000的全部关闭然后编译
>
> ```shell
> make ARCH=arm64 LLVM=1 -j8
> ```



首先编译我们的驱动

```shell
cd e1000-driver/src/linux
make KDIR=/mnt/rust-for-linux/linux/build
```



这里我们直接使用debian的镜像也就是[4.2中的方法2](#4.2)

```shell
qemu-system-aarch64 -machine virt -cpu cortex-a57 -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -nographic -append "root=LABEL=rootfs console=ttyAMA0" -initrd initrd -device e1000,netdev=net0,bus=pcie.0 -netdev user,id=net0 -kernel ../arch/arm64/boot/Image
```



```shell
ip address
```

我们可以看到有

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
```

eth0 网卡

这时候我们可以用scp 把编译的ko文件复制进来

```
scp xxx@ip:/mnt/rust-for-linux/e1000-driver/src/linux/../e1000_for_linux.ko /root/
```

然后禁用我们的eth0

```shell 
ip link set eth0 down
```

加载我们用rust写的e1000的驱动

```shell
insmod e1000_for_linux.ko
```

这时候执行`ip l`会发现多了一个eth1

启动eth1

```shell
ip link set eth1 up
```

由于没有分配ip所以手动分配ip

因为qemu网关是10.0.2.2

所以这里分配

```shell
ip addr add 10.0.2.20/24 dev eth1
ip route add default via 10.0.2.2 dev eth1
```

之后执行ping

```shell
ping 10.0.2.2
```

可以看到如下打印

```text
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
[  336.638258] rust_e1000dev: start xmit
[  336.638671] rust_e1000dev: Read E1000_TDT = 0x0
[  336.638733] rust_e1000dev: >>>>>>>>> TX PKT 60
[  336.638844] rust_e1000dev:
[  336.638844]
[  336.639136] rust_e1000dev: handle_irq
[  336.639289] rust_e1000dev: irq::Handler E1000_ICR = 0x83
[  336.639680] rust_e1000dev: NapiPoller poll
[  336.639777] rust_e1000dev: Read E1000_RDT + 1 = 0x0
[  336.639814] rust_e1000dev: RX PKT 64 <<<<<<<<<
[  336.640057] rust_e1000dev: e1000_recv
[  336.640057]
[  336.640605] rust_e1000dev: handle_rx_irq 1 packets, 64 bytes
[  336.641801] rust_e1000dev: start xmit
[  336.641883] rust_e1000dev: Read E1000_TDT = 0x1
[  336.641890] rust_e1000dev: >>>>>>>>> TX PKT 98
[  336.641972] rust_e1000dev:
[  336.641972]
[  336.642191] rust_e1000dev: handle_irq
[  336.642366] rust_e1000dev: irq::Handler E1000_ICR = 0x83
[  336.646625] rust_e1000dev: NapiPoller poll
[  336.646726] rust_e1000dev: Read E1000_RDT + 1 = 0x1
[  336.646734] rust_e1000dev: RX PKT 98 <<<<<<<<<
[  336.646835] rust_e1000dev: e1000_recv
[  336.646835]
[  336.647052] rust_e1000dev: handle_rx_irq 1 packets, 98 bytes
64 bytes from 10.0.2.2: icmp_seq=1 ttl=255 time=11.5 ms
[  337.641037] rust_e1000dev: start xmit
[  337.641723] rust_e1000dev: Read E1000_TDT = 0x2
[  337.641802] rust_e1000dev: >>>>>>>>> TX PKT 98
[  337.642021] rust_e1000dev:
[  337.642021]
[  337.642528] rust_e1000dev: handle_irq
[  337.649010] rust_e1000dev: irq::Handler E1000_ICR = 0x83
[  337.650481] rust_e1000dev: NapiPoller poll
[  337.650852] rust_e1000dev: Read E1000_RDT + 1 = 0x2
[  337.650886] rust_e1000dev: RX PKT 98 <<<<<<<<<
[  337.651381] rust_e1000dev: e1000_recv
[  337.651381]
[  337.651737] rust_e1000dev: handle_rx_irq 1 packets, 98 bytes
64 bytes from 10.0.2.2: icmp_seq=2 ttl=255 time=13.1 ms
```

完成了我们的e1000网卡的checkpoint编写
