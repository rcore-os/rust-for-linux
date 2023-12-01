---
Title: Rust for Linux驱动模块开发报告-童浩轩
Date: 2023-11
Tags:
    - Author: 童浩轩
    - Reports Repo: https://github.com/Tunghohin/rust-for-linux/tree/main/reports
---

## 编译 linux

#### 1. 获取源码

```sh
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth=1
```

#### 2. 安装依赖

- 获取依赖包

```sh
sudo apt-get -y install \
  binutils build-essential libtool texinfo \
  gzip zip unzip patchutils curl git \
  make cmake ninja-build automake bison flex gperf \
  grep sed gawk bc \
  zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev libelf-dev libssl-dev
```

- 获取 clang 和 llvm

```sh
sudo apt install clang llvm
```

#### 3. 编译 linux 内核

```sh
make ARCH=arm64 LLVM=1 defconfig
make ARCH=arm64 LLVM=1 menuconfig # general setup 里打开 rust support
cd ./build
bear -- make ARCH=arm64 LLVM=1 -j12 # 顺便生成 compile_commands.json
```

#### 4. 获取 dqib 系统镜像

```sh
https://people.debian.org/~gio/dqib/
```

使用脚本运行 qemu

```sh
#!/bin/bash

qemu-system-aarch64 \
	-machine 'virt' \
	-cpu 'cortex-a57' \
	-m 1G \
	-device virtio-blk-device,drive=hd \
	-drive file=image.qcow2,if=none,id=hd \
	-device virtio-net-device,netdev=net \
	-netdev user,id=net,hostfwd=tcp::2222-:22 \
	-kernel /home/notaroo/kernel_dev/linux-rust/build/arch/arm64/boot/Image.gz \
	-initrd initrd \
	-append "root=LABEL=rootfs console=ttyAMA0"
```

## 运行结果

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/imgs/Screenshot from 2023-11-08 13-11-58.png">

## 加载模块

将 .ko 拷贝到制作好的 rootfs 里
运行

```sh
insmod rust_hello.ko
```

## 碰到的问题

#### 1. 根文件系统构建

安装 debootstrap 和 qemu-utils

```sh
sudo apt-get install debootstrap qemu-utils
```

创建磁盘镜像文件格式化并挂载

```sh
qemu-img create -f raw rootfs.img 4G
mkfs.ext4 rootfs.img
sudo mkdir /mnt/rootfs
sudo mount -o loop rootfs.img /mnt/rootfs
```

安装 debian 镜像

```sh
sudo debootstrap --arch arm64 stable /mnt/rootfs https://mirrors.tuna.tsinghua.edu.cn/debian/
```

运行 `run.sh`

#### 2. rust-analyzer

```sh
make LLVM=1 ARCH=arm64 rust-analyzer
```

#### 3. deboostrap 下载慢

加上清华的源

```
https://mirrors.tuna.tsinghua.edu.cn/debian/
```

#### 4. 根文件系统挂载不上

在根文件创建 /dev/vda，在 qemu 中指定 root=/dev/vda rw

#### 5. qemu 连不上网

配置 `/etc/network/interfaces`

```
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp
```

同时在 qemu 里根文件指定路径后添加 rw

## 运行结果

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/imgs/Screenshot from 2023-11-09 10-10-02.png">

## binding

#### 1.在 include/linux 中创建文件

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/Screenshot from 2023-11-18 00-55-12.png">

#### 2.在 rust/bindings/bindings_helper.h 中 include 头文件，并将其于 /rust/helpers.c 中 export

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/Screenshot from 2023-11-18 00-54-12.png" width=60%>
<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/Screenshot from 2023-11-18 00-54-50.png" width=60%>

#### 3.通过 rust 模块调用

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/Screenshot from 2023-11-18 00-53-33.png">

## 要点

#### dma

DMA（Direct Memory Access）可以使得外部设备可以不用 CPU 干预，直接把数据传输到内存，这样可以解放 CPU，提高系统性能

使用 dma_alloc_coherent/dma_free_coherent，分配/释放网卡 tx_ring 和 rx_ring 的 dma 空间

#### napi

由于当网络包首发密集时，频繁地触发中断会严重影响 cpu 的执行效率，napi 是 linux 上采用的以提高网络处理效率的技术。
网卡接收到数据，通过硬中断通知 CPU 进行处理，但是当网卡有大量数据涌入时，频繁中断消耗大量的 CPU 处理时间在中断自身处理上，使得网卡和 CPU 工作效率低下，所以系统采用了硬中断进入轮询队列 + 软中断退出轮询队列技术，提升数据接收处理效率

#### ring buffer

e1000 网卡环形缓冲区，通过 DMA 映射到内存

#### sk_buff

网络是分层的，对于应用层而言不用关心具体的底层是如何工作的，只需要按照协议将要发送或接收的数据打包成 packet 即可。

要使用 sk_buff 必须先分配
使用 alloc_skb_ip_align 分配一个 sk_buff

接收数据时，将网卡 rx_ring 受到的 packet 写入到 sk_buff
发送数据时，将 sk_buff 的数据使用 e1000_transmit 发送到网卡的 tx_ring

#### 参考资料

https://www.zhihu.com/people/wenfh2020/posts?page=2

https://stackoverflow.com/questions/47450231/what-is-the-relationship-of-dma-ring-buffer-and-tx-rx-ring-for-a-network-card?answertab=votes#tab-top

https://github.com/fujita/rust-e1000

https://elixir.bootlin.com/linux/v4.19.121/source/drivers/net/ethernet/intel/e1000
