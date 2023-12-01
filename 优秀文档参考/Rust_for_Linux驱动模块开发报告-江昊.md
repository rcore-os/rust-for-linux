---
Title: Rust for Linux驱动模块开发报告-江昊
Date: 2023-11
Tags:
    - Author: 江昊
    - Reports Repo: https://github.com/xxkeming/rust/blob/main/rust-for-linux.md
---

## 练习 1,2 作业

### 源码下载
```
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth 1
git clone https://github.com/fujita/linux.git -b rust-e1000 --depth 1
```

### 环境搭建
```
## Install dependency packages
sudo apt-get -y install \
  binutils build-essential libtool texinfo \
  gzip zip unzip patchutils curl git \
  make cmake ninja-build automake bison flex gperf \
  grep sed gawk bc \
  zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev libelf-dev libssl-dev

## Install the LLVM
sudo apt-get install clang-format clang-tidy clang-tools clang clangd libc++-dev libc++1 libc++abi-dev libc++abi1 libclang-dev libclang1 liblldb-dev libllvm-ocaml-dev libomp-dev libomp5 lld lldb llvm-dev llvm-runtime llvm python3-clang

## Add Rust environment
cd linux
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
make LLVM=1 rustavailable
```

### 编译时遇到的问题 bindgen版本过高,修改了部分增对bindgen参数的Makefile配置
<img width="1633" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/3a04cfab-e44a-4f5a-9198-f983c7ba8ea4">

### 编译流程,配置开启rust,并把练习2的代码及配置加入
```
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig

cd build
make ARCH=arm64 LLVM=1 -j4
```

### 启动方式
```
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel /home/parallels/linux/build/arch/arm64/boot/Image.gz -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"

```

### 运行结果
<img width="1197" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/73cae450-bf36-4d90-acdb-a0b61fd636d6">


## 练习 3,4 作业
### 网络环境配置
```
主机配置:
sudo ip link add br0 type bridge
sudo ip addr add 192.168.100.50/24 brd 192.168.100.255 dev br0
sudo ip tuntap add mode tap user $(whoami)
ip tuntap show
sudo ip link set tap0 master br0
sudo ip link set dev br0 up
sudo ip link set dev tap0 up

虚拟机配置:
ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
ifconfig eth0 192.168.100.224 netmask 255.255.255.0 broadcast 192.168.100.255 up

qemu启动方式:
qemu-system-aarch64 -M virt -cpu cortex-a57 -smp 1 -m 256M -kernel build/arch/arm64/boot/Image.gz -initrd ../busybox-1.36.1/initramfs.cpio.gz -nographic -vga none -no-reboot -append "init=/init console=ttyAMA0" -netdev tap,ifname=tap0,id=tap0,script=no,downscript=no -device e1000,netdev=tap0
```
### 添加自定义函数
```
1.直接在rust/helpers.c添加自定义函数void rust_helper_test(void)
2.在rust/bindings/lib.rs对bindings_helper添加pub
3.调用方式:kernel::bindings::bindings_helper::test()
```
### e100网卡驱动作业仓库地址及运行结果
```
https://github.com/xxkeming/e1000-driver
```
<img width="1551" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/ead8e636-3ae8-45e6-924f-e55ad3cf0ff5">

## 补充: 自定义基本模块
### test.rs
```
use kernel::prelude::*;
module! {
  type: RustTest,
  name: "rust_test",
  author: "keming",
  description: "test module in rust",
  license: "GPL",
}
struct RustTest {}
impl kernel::Module for RustTest {
  fn init(_name: &'static CStr, _module: &'static ThisModule) -> Result<Self> {
      pr_info!("test from Rust module");
      Ok(RustTest {})
  }
}
```
### makefile
```
obj-m := test.o
PWD := $(shell pwd)
ARCH ?= arm64
KDIR ?= /lib/modules/$(shell uname -r)/build
default:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ modules
clean:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ clean
```
