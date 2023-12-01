---
Title: Rust for Linux驱动模块开发报告-杨凯
Date: 2023-11
Tags:
    - Author: 杨凯
    - Reports Repo: https://github.com/guevaraya/rust-for-linux/
---

# 学习记录

### 问题1
代码下载，由于rust-for-linux的官方代码全镜像约为4G，国内网络想下载下来是很难的，因此需要添加--depth=1 并指定分支rust-dev，但是太慢
最终还是用下面命令完成下载：

```
git clone https://github.com/fujita/linux.git -b rust-e1000

```

由于问题3，最后还是从官网下
git clone <https://github.com/Rust-for-Linux/linux> -b rust-dev --depth=1

### 问题2

根据脚本 scripts/min-tool-version.sh分析
rustup override set $(scripts/min-tool-version.sh rustc)  相当于 rustup override set  1.62.0 
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli相当于
cargo install --locked --version 0.56.0 bindgen-cli 安装

### 问题3

menuconfig搞了半天发现没有找到所谓的rust support选项，经过对比config.in发现

```
rust_is_available=n，
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
    Blocking waiting for file lock on package cache
    Updating `ustc` index

vierror: could not find `bindgen-cli` in registry `crates-io` with version `=0.56.0`

```

根因还是bindgen没有安装好，在crates-io官网搜索bindgen，确实最新的版本是0.65.1，但是fujita下载的版本太低，适应官网地址编译，解决了问题

```
cat ~/.cargo/config
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
# registry = "https://mirrors.ustc.edu.cn/crates.io-index"
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
[http]
check-revoke = false

```

配置完成结果
![image](https://github.com/guevaraya/rust-for-linux/assets/446973/e9c21606-476f-4ca1-9b77-63b2f87fe9e7)

## 练习2 自定义编写Rust内核驱动模块

* 1. linux目录下samples/rust，新建rust_helloworld.rs  并添加对应的Kconfig
     **rust_helloworld.rs** 


```
// SPDX-License-Identifier: GPL-2.0
//! Rust minimal sample.
      
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
  fn init(_name: &'static CStr, _module: &'static ThisModule) -> Result<Self> {
      pr_info!("Hello World from Rust module");
      Ok(RustHelloWorld {})
  }
}

```

* 1. samples/rust/Kconfig 配置修改如下：


```
obj-$(CONFIG_SAMPLE_RUST_FS)            += rust_fs.o
obj-$(CONFIG_SAMPLE_RUST_SELFTESTS)        += rust_selftests.o
      // ++++++++++++++++ add here
obj-$(CONFIG_SAMPLE_RUST_HELLOWORLD)        += rust_helloworld.o
      // ++++++++++++++++
subdir-$(CONFIG_SAMPLE_RUST_HOSTPROGS)        += hostprogs
  
config SAMPLE_RUST_MINIMAL
  tristate "Minimal"
  help
    This option builds the Rust minimal module sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_minimal.
      
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
config SAMPLE_RUST_PRINT
  tristate "Printing macros"
  help
    This option builds the Rust printing macros sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_print.
      
    If unsure, say N.

```

3，配置使能 run make LLVM=1 menuconfig

```
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <*>Print Helloworld in Rust (NEW)

```

4，编译内核 run make  LLVM=1

5, 启动QEMU,

为了方便直接在<https://people.debian.org/~gio/dqib/> 这里下载根文件系统，解压后参照readme.txt，更换自己的内核后，运行QEMU目录：

```
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel ../linux/build/arch/arm64/boot/Image -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"

```

### 使用 busybox 制作内存文件系统 initramfs：

1. 下载解压 busybox 并配置环境变量

   ```shell
   wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
   tar -xf busybox-1.35.0.tar.bz2
   cd busybox-1.35.0
   # 配置环境变量
   export ARCH=arm64
   export CROSS_COMPILE=aarch64-linux-gnu-
   ```

   如果教程工具没有安装，请用下面命令


```
sudo apt-get install gcc-aarch64-linux-gnu

```

1. 配置编译内核的参数

   ```shell
   # busybox-1.35.0目录下
   make menuconfig
   # 修改配置，选中如下项目，静态编译
   # Settings -> Build Options -> [*] Build static binary (no share libs)

   ```


1. 编译

   ```shell
   make -j `nproc`

   ```


编译头文件出错，错误如下：
```
make -j `nproc`
  SPLIT   include/autoconf.h -> include/config/*
  GEN     include/bbconfigopts.h
  GEN     include/common_bufsiz.h
  GEN     include/embedded_scripts.h
  HOSTCC  applets/usage
  HOSTCC  applets/applet_tables
  GEN     include/usage_compressed.h
  GEN     include/applet_tables.h include/NUM_APPLETS.h
  GEN     include/applet_tables.h include/NUM_APPLETS.h
  HOSTCC  applets/usage_pod
  CC      applets/applets.o
In file included from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/limits.h:195,
                 from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/syslimits.h:7,
                 from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/limits.h:34,
                 from include/platform.h:157,
                 from include/libbb.h:13,
                 from include/busybox.h:8,
                 from applets/applets.c:9:
/usr/include/limits.h:26:10: fatal error: bits/libc-header-start.h: 没有那个文件或目录
   26 | #include <bits/libc-header-start.h>
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.build:198：applets/applets.o] 错误 1
make: *** [Makefile:372：applets_dir] 错误 2

```

需要安装apt install gcc-multilib

```
 make
  CC      applets/applets.o
In file included from /usr/include/bits/errno.h:26,
                 from /usr/include/errno.h:28,
                 from include/libbb.h:17,
                 from include/busybox.h:8,
                 from applets/applets.c:9:
/usr/include/linux/errno.h:1:10: fatal error: asm/errno.h: 没有那个文件或目录
    1 | #include <asm/errno.h>
      |          ^~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.build:198：applets/applets.o] 错误 1
make: *** [Makefile:372：applets_dir] 错误 2

```

解决方法：sudo ln -s /usr/i686-linux-gnu/include/asm /usr/include/asm

1. 安装

   安装前 busybox-1.35.0 目录下的文件如下图所示

输入如下命令


```shell
make install
```


安装后目录下生成了\_install 目录


1. 在\_install 目录下创建后续所需的文件和目录

   ```shell
   cd _install
   mkdir proc sys dev tmp
   touch init
   chmod +x init

   ```

2. 用任意的文本编辑器编辑 init 文件内容如下

   ```shell
   #!/bin/sh

   # 挂载一些必要的文件系统
   mount -t proc none /proc
   mount -t sysfs none /sys
   mount -t tmpfs none /tmp
   mount -t devtmpfs none /dev

   ```


```
# 停留在控制台
exec /bin/sh
```



1. 用 busybox 制作 initramfs 文件

   ```shell
   # _install目录
   find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz

   ```

   执行成功后可在 busybox-1.35.0 目录下找到 initramfs.cpio.gz 文件

2. 进入 qemu 目录下，执行如下命令

   
   ```shell
   qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel (your Image path) -initrd (your initramfs.cpio.gz path) -nographic -append "init=/init console=ttyAMA0"
   ```

   其中`(your Image path)`为上一个任务中最后的`Image`镜像文件所在的目录，`(your initramfs.cpio.gz)`为步骤 7 中执行成功后得到的 initramfs.cpio.gz 的目录，例如

   ```shell
   qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel /home/jun/maodou/linux/arch/arm64/boot/Image -initrd /home/jun/maodou/busybox-1.35.0/initramfs.cpio.gz -nographic -append "init=/init console=ttyAMA0"
   ```

   

   * 问题：failed to find romfile "efi-virtio.rom"
   解决：apt-get install ipxe-qemu
       
![image](https://github.com/guevaraya/rust-for-linux/assets/446973/c2fc4220-df5f-4353-a07d-79381223d87d)

[\[1\] QEMU安装启动实例](http://iric.tpddns.cn:9955/#/Rust/2023%E7%A7%8B%E5%86%AC%E8%AE%AD%E7%BB%83%E8%90%A5/%E7%AC%AC%E4%B8%89%E9%98%B6%E6%AE%B5/rust-for-linux/Qemu%E6%A8%A1%E6%8B%9F%E5%90%AF%E5%8A%A8/README)

[\[2\] 编译 linux for rust 并制作 initramfs 最后编写 rust_helloworld 内核驱动 并在 qemu 环境加载](https://blog.csbxd.fun/archives/1699538198511)
[通过 QEMU 打开学习 Linux kernel 的新世界](https://www.jianshu.com/p/9b68e9ea5849)


## 练习3& 4 编写bindling 和填充checkpoint编写E1000驱动

作业描述：<https://github.com/rcore-os/rust-for-linux/blob/main/exercise3.md> 

参考链接：  https://github.com/yuoo655/e1000-driver.git
### bingding填写
在linux 的rust驱动中需要调用内核API，需要借助bindgen是一个将C 语言的头文件转化为rust源码的工具，才能调用。https://rust-lang.github.io/rust-bindgen/
下面是一个C的头文件

linux/include/r4l_sample.h
```
int echo_for_rust(const unsigned char * str)
{
        printk("[r4l test]:%s\n", str);
        return 0;
}
EXPORT_SYMBOL_GPL(echo_for_rust);
```
linux/rust/helpers.c
````
int rust_helper_echo(const unsigned char * str)
{
        return echo_for_rust(str);
}
EXPORT_SYMBOL_GPL(rust_helper_echo);
````
这样就可以在rust里面通过bingling::echo调用C代码了

### 驱动编译和文件系统制作的Makefile脚本
```
all: run

menuconfig:
        #make -C linux ARCH=arm64 LLVM=1 O=build menuconfig
        make -C linux-fujita ARCH=arm64 LLVM=1 O=build menuconfig
build:
        #make -C linux/build ARCH=arm64 LLVM=1 -j4
        make -C linux-fujita/build ARCH=arm64 LLVM=1 -j4
run:
        #qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel linux/build/arch/arm64/boot/Image -initrd busybox-1.33.2/initramfs.cpio.gz -nographic -append "init=/linuxrc console=ttyAMA0"
        qemu-system-aarch64 -M virt \
        -cpu cortex-a72 \
        -smp 8 \
        -m 128M \
        -device virtio-net-device,netdev=net  \
        -netdev user,id=net,hostfwd=tcp::2222-:22  \
        -kernel linux-fujita/build/arch/arm64/boot/Image \
        -initrd busybox-1.33.2/initramfs.cpio.gz \
        -nographic \
        -device e1000,netdev=net0,bus=pcie.0 \
        -netdev user,id=net0 \
        -append "init=/linuxrc console=ttyAMA0"
run_initrd:
        qemu-system-aarch64 -machine 'virt' \
        -cpu 'cortex-a57'  \
        -m 1G -device virtio-blk-device,drive=hd  \
        -drive file=dqib_arm64-virt/image.qcow2,if=none,id=hd  \
        -device virtio-net-device,netdev=net  \
        -netdev user,id=net,hostfwd=tcp::2222-:22  \
        -kernel linux/build/arch/arm64/boot/Image \
        -initrd dqib_arm64-virt/initrd \
        -nographic  \
        -append "root=LABEL=rootfs console=ttyAMA0"
busybox:
        export ARCH=arm64
        export CROSS_COMPILE=aarch64-linux-gnu-
        make -C busybox-1.33.2
e1000:
        make -C e1000-driver/src/linux
setup: e1000
        rm  busybox-1.33.2/_install/e1000_for_linux.ko
        cp  e1000-driver/src/e1000_for_linux.ko   busybox-1.33.2/_install/
        cd busybox-1.33.2/; ./setup.sh; cd -

```
### 进入状态

刚开始老师讲解e1000驱动过程，云里雾里，驱动作业无从下手的感觉，前期内核编译和文件系统制作原本很简单，但是各种安装包不兼容问题，并且rust-for-linux的驱动和内核接口变化太快，导致驱动调试，e1000-driver最好还是要基于fujita-linux编译会更好一点，这块我遇到的问题是

先用官网的<https://github.com/Rust-for-Linux/linux的编译并制作好了文件系统，helloworld驱动也验证通过，调试> https\://github.com/yuoo655/e1000-driver.git驱动的时候发现其与<https://github.com/Rust-for-Linux/linux的rust-dev分支匹配起来有很多编译问题，>

随转向 编译<https://github.com/fujita/linux.git，但bindgen要求的版本比较低，结果硬着用新bindgen> 1.62.0 替换1.56.0 编译过去，修改了内核部分代码，花费了三四天才真正开始调驱动，看起其他同学都已经调试完成了，心里还是有点焦急的。但没办法，硬着头皮调试，不能放弃。

### 驱动调试网络原理困惑

由于之前没有接触过网卡驱动，对网卡配置有点手足无措，最后经过调查E1000有fujita的日本人已经调试好的代码参考 <https://github.com/fujita/rust-e1000，有前辈的代码，可以摸着它的代码调试了>

![image](https://github.com/guevaraya/rust-for-linux/assets/446973/c112bccc-2c47-4b8c-abfc-a0d59e5684de)

驱动OSI 7层协议，应用层，表示层，会话层，传输层，网络层，链路层和物理层
物理层和链路层对应网络的驱动，应用层，表示层，会话层对应ftp，http，ssh等，而传输层就是UDP，TCP，以及最热门的QUIC协议，而链路层典型的是MAC层，物理层是Phy层，IDH等。

代码经过一系列的折腾和编译，也参考了<https://github.com/lispking/rust-e1000-driver> 这位同学的代码，启动之后结果发现没有probe打印，


经过仔细对吧，我们的开机启动qemu没有加入e1000网卡参数
处理中断的时候没有考虑None的情况，导致panic

```
 -device e1000,netdev=net0,bus=pcie.0 \
        -netdev user,id=net0 \

```

### 网卡数据发送：

HTTP发送网络报过程

套接字缓冲区查询命令ss -ntt



看来start_xmit从网络层项链路层发送skb_buffer数据到tx_ring

skb_buffer 数据结构

Ring buffer结构如下：

![Uploading image.png…]()

e1000 收发参考原理：<https://blog.csdn.net/u010180372/article/details/119525638>

\[内核源码] Linux 网络数据接收流程（TCP）- NAPI : <https://zhuanlan.zhihu.com/p/452612386> 

### 渐入佳境-驱动debug

* 问题： eth1 up的时候内核崩溃

```
/ # ifconfig eth1 up
[   31.261677] rust_e1000dev: Ethernet E1000 open
[   31.262660] rust_e1000dev: New E1000 device @ 0xffff800008480000
[   31.268236] rust_e1000dev: Allocated vaddr: 0xffff559803193000, paddr: 0x43193000
[   31.271569] rust_e1000dev: Allocated vaddr: 0xffff5598033d0000, paddr: 0x433d0000
[   31.282618] rust_e1000dev: Allocated vaddr: 0xffff559805c80000, paddr: 0x45c80000
[   31.289265] rust_e1000dev: Allocated vaddr: 0xffff559805d00000, paddr: 0x45d00000
[   31.292126] rust_e1000dev: e1000 CTL: 0x140240, Status: 0x80080783
[   31.293155] rust_e1000dev: e1000_init has been completed
[   31.294449] rust_e1000dev: e1000 device is initialized
[   31.302014] rust_e1000dev: handle_irq
[   31.303574] rust_e1000dev: irq::Handler E1000_ICR = 0x4
[   31.307389] rust_e1000dev: NapiPoller poll
[   31.307501] rust_e1000dev: get stats64
[   31.309985] rust_e1000dev: e1000_recv
[   31.309985]
[   31.310547] rust_kernel: panicked at 'called `Option::unwrap()` on a `None` value', /srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../e1000_for_linux.rs:86:54
[   31.321218] ------------[ cut here ]------------
[   31.322559] rust_e1000dev: get stats64
[   31.324359] kernel BUG at rust/helpers.c:48!
[   31.326392] Internal error: Oops - BUG: 00000000f2000800 [#1] PREEMPT SMP
[   31.329246] Modules linked in: e1000_for_linux(O)
[   31.335888] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G           O       6.1.0-rc1-g5fc95830739f-dirty #2
[   31.340642] Hardware name: linux,dummy-virt (DT)
[   31.343687] pstate: 60000005 (nZCv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[   31.346427] pc : rust_helper_BUG+0x4/0x8
[   31.350033] lr : rust_begin_unwind+0x5c/0x60
[   31.352394] sp : ffff800008003ca0
[   31.354242] x29: ffff800008003cf0 x28: ffff5598033d0000 x27: ffff800008003de0
[   31.358153] x26: ffff800008003ef0 x25: ffff800008003f00 x24: ffffb7c2d17ca5fd
[   31.361537] x23: ffffb7c31aea4634 x22: 0000000000000001 x21: 0000000000000100
[   31.364296] x20: 0000000000000000 x19: ffff800008003e00 x18: ffffffffffffffff
[   31.366870] x17: 0000000000000041 x16: ffffb7c31adfd5b8 x15: 0000000000000004
[   31.369318] x14: 0000000000000fff x13: ffffb7c31b95b090 x12: 0000000000000003
[   31.372159] x11: 00000000ffffefff x10: c0000000ffffefff x9 : 306c1c7bb7800c00
/ # [   31.374389] x8 : 306c1c7bb7800c00 x7 : 205d373435303133 x6 : 332e31332020205b
[   31.378287] x5 : ffffb7c31bcf7eaf x4 : 0000000000000001 x3 : 0000000000000000
[   31.380756] x2 : 0000000000000000 x1 : ffff800008003a60 x0 : 00000000000000e3
[   31.383739] Call trace:
[   31.384726]  rust_helper_BUG+0x4/0x8
[   31.386219]  _RNvNtCs3yuwAp0waWO_4core9panicking9panic_fmt+0x34/0x38
[   31.389057]  _RNvNtCs3yuwAp0waWO_4core9panicking5panic+0x38/0x3c
[   31.390407]  _RNvXs2_CsejK4Ne9WL8c_15e1000_for_linuxNtB5_6PollerNtNtCsfATHBUcknU9_6kernel3net10NapiPoller4poll+0x560/0x564 [e1000_for_linux]
[   31.396475]  _RNvMs7_NtCsfATHBUcknU9_6kernel3netINtB5_11NapiAdapterNtCsejK4Ne9WL8c_15e1000_for_linux6PollerE13poll_callbackBR_+0x44/0x58 [e1000_for_linux]
[   31.400059]  __napi_poll+0x48/0x1cc
[   31.401478]  net_rx_action+0x134/0x2d4
[   31.402281]  __do_softirq+0xdc/0x25c
[   31.403612]  ____do_softirq+0x10/0x1c
[   31.404441]  call_on_irq_stack+0x2c/0x54
[   31.405413]  do_softirq_own_stack+0x1c/0x28
[   31.406320]  __irq_exit_rcu+0x98/0xec
[   31.407628]  irq_exit_rcu+0x10/0x1c
[   31.408937]  el1_interrupt+0x8c/0xbc
[   31.409942]  el1h_64_irq_handler+0x18/0x24
[   31.411241]  el1h_64_irq+0x64/0x68
[   31.412316]  arch_cpu_idle+0x18/0x28
[   31.413007]  cpuidle_idle_call+0x6c/0x1e4
[   31.413789]  do_idle+0xcc/0xf4
[   31.414408]  cpu_startup_entry+0x24/0x28
[   31.416156]  kernel_init+0x0/0x1a0
[   31.417162]  start_kernel+0x0/0x41c
[   31.418206]  start_kernel+0x328/0x41c
[   31.419322]  __primary_switched+0xbc/0xc4
[   31.421651] Code: a8c17bfd d50323bf d65f03c0 d503233f (d4210000)
[   31.425128] ---[ end trace 0000000000000000 ]---
[   31.427231] Kernel panic - not syncing: Oops - BUG: Fatal exception in interrupt
[   31.428962] SMP: stopping secondary CPUs
[   31.432119] Kernel Offset: 0x37c311e00000 from 0xffff800008000000
[   31.433208] PHYS_OFFSET: 0xffffaa6840000000
[   31.433966] CPU features: 0x20000,2013c080,0000421b
[   31.435508] Memory Limit: none
[   31.437214] ---[ end Kernel panic - not syncing: Oops - BUG: Fatal exception in interrupt ]---

```

边看老师的视频，参考fujita和查找网络资料，终于ping通了

```
qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel linux/build/arch/arm64/boot/Image -initrd busybox-1.33.2/initramfs.cpio.gz -nographic -append "init=/linuxrc console=ttyAMA0"
\-device virtio-net-device,netdev=net  \\
\-netdev user,id=net,hostfwd=tcp::2222-:22  \\
\-device e1000,netdev=net0,bus=pcie.0 \\
\-netdev user,id=net0 \\

\[    0.000000] Booting Linux on physical CPU 0x0000000000 \[0x410fd083]

\[    0.000000] Linux version 6.1.0-rc1-g5fc95830739f-dirty (root@openmediavault) (Debian clang version 11.0.1-2, LLD 11.0.1) #4 SMP PREEMPT Sat Nov 18 17:07:27 CST 2023

\[    0.000000] random: crng init done

\[    0.000000] Machine model: linux,dummy-virt

...

/ # ping 10.0.2.2

PING 10.0.2.2 (10.0.2.2): 56 data bytes

\[   18.279362] \[r4l test]:binding start xmit

\[   18.281663] rust_e1000dev: Read E1000_TDT = 0x0

\[   18.282078] rust_e1000dev: >>>>>>>>> TX PKT 60

\[   18.282844] rust_e1000dev:

\[   18.282844]

\[   18.284086] rust_e1000dev: handle_irq

\[   18.291407] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   18.293517] rust_e1000dev: NapiPoller poll

\[   18.295755] rust_e1000dev: Read E1000_RDT + 1 = 0x0

\[   18.296219] rust_e1000dev: RX PKT 64 \<\<\<\<\<\<\<\<\<

\[   18.299517] rust_e1000dev: e1000_recv

\[   18.299517]

\[   18.301622] rust_e1000dev: irq::Handler recv = 0x1

\[   18.316187] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x40

\[   18.330359] \[r4l test]:binding start xmit

\[   18.332177] rust_e1000dev: Read E1000_TDT = 0x1

\[   18.332326] rust_e1000dev: >>>>>>>>> TX PKT 98

\[   18.334101] rust_e1000dev:

\[   18.334101]

\[   18.338608] rust_e1000dev: handle_irq

\[   18.340693] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   18.346517] rust_e1000dev: NapiPoller poll

\[   18.347330] rust_e1000dev: Read E1000_RDT + 1 = 0x1

\[   18.347421] rust_e1000dev: RX PKT 98 \<\<\<\<\<\<\<\<\<

\[   18.348343] rust_e1000dev: e1000_recv

\[   18.348343]

\[   18.350169] rust_e1000dev: irq::Handler recv = 0x1

\[   18.353030] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x62

64 bytes from 10.0.2.2: seq=0 ttl=255 time=122.904 ms

```
