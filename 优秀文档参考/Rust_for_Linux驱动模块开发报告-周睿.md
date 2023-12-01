---
Title: Rust for Linux驱动模块开发报告-周睿
Date: 2023-11
Tags:
    - Author: 周睿
    - Reports Repo: https://github.com/ZR233/learn_rust_for_linux
---

# Learn Rust-for-Linux

## Day 1

今天的目标是编译并在`Qemu`上运行`kernel`。
源码相当大，有4.5G，还需要给生成留出空间，感觉至少得给20G。
编译`Qemu`

```shell
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev libaio-dev libbluetooth-dev libbrlapi-dev libbz2-dev libnfs-dev libiscsi-dev ninja-build python3 python3-pip python3-venv

pip install sphinx sphinx-rtd-theme


git clone git@github.com:qemu/qemu.git
cd qemu
mkdir build
cd build
../configure --target-list=riscv64-softmmu,riscv64-linux-user,aarch64-softmmu,aarch64-linux-user,x86_64-softmmu,x86_64-linux-user
make -j4
sudo make install 
```

编译`BusyBox`：

```shell
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xvf busybox-1.36.1.tar.bz2 
cd busybox-1.36.1/
make  ARCH=arm64 menuconfig -j4
make  ARCH=arm64 install -j4

```

基于BusyBox制作initrd
制作`initrd`步骤略多。这里`make_initrd.sh`就是用来自动制作`initrd`的脚本，内容：
```shell
#!/bin/bash

MOUNT_DIR=mnt
CURR_DIR=`pwd`

rm initrd.ext4
dd if=/dev/zero of=initrd.ext4 bs=1M count=32
mkfs.ext4 initrd.ext4

mkdir -p $MOUNT_DIR
mount initrd.ext4 $MOUNT_DIR
cp -arf busybox-1.35.0/_install/* $MOUNT_DIR

cd $MOUNT_DIR
mkdir -p etc dev mnt proc sys tmp mnt etc/init.d/

echo "proc /proc proc defaults 0 0" > etc/fstab
echo "tmpfs /tmp tmpfs defaults 0 0" >> etc/fstab
echo "sysfs /sys sysfs defaults 0 0" >> etc/fstab

echo "#!/bin/sh" > etc/init.d/rcS
echo "mount -a" >> etc/init.d/rcS
echo "mount -o remount,rw /" >> etc/init.d/rcS
echo "echo -e \"Welcome to ARM64 Linux\"" >> etc/init.d/rcS
chmod 755 etc/init.d/rcS

echo "::sysinit:/etc/init.d/rcS" > etc/inittab
echo "::respawn:-/bin/sh" >> etc/inittab
echo "::askfirst:-/bin/sh" >> etc/inittab
chmod 755 etc/inittab

cd dev
mknod console c 5 1
mknod null c 1 3
mknod tty1 c 4 1

cd $CURR_DIR
umount $MOUNT_DIR
echo "make initrd ok!"

```

然后必须 `sudo` 执行 `sudo ./make_inird.sh`

接着，`Qemu`启动！

```shell
qemu-system-aarch64 \
    -nographic \
    -M virt \
    -cpu cortex-a57 \
    -smp 2 \
    -m 4G \
    -kernel /mnt/sdb/dev/linux/build/arch/arm64/boot \
    -append "nokaslr root=/dev/ram init=/linuxrc console=ttyS0" \
    -initrd disk.img
```

![image](https://github.com/ZR233/learn_rust_for_linux/blob/main/image/qemu1.png)

果然第一次就失败了。似乎是文件系统没做对

## Day2

仔细查看`qemu-system-aarch64 -h`，发现指令 `-initrd file    use 'file' as initial ram disk` 将文件挂载为ram，而默认`kernel`未添加`ram`支持，应该改为挂载`-hda/-hdb file  use 'file' as hard disk 0/1 image`，所以运行命令修改为
```shell
#!/bin/bash

qemu-system-aarch64 \
    -nographic \
    -M virt \
    -cpu cortex-a57 \
    -smp 2 \
    -m 4G \
    -kernel /mnt/sdb/dev/linux/build/arch/arm64/boot/Image \
    -append "nokaslr root=/dev/sda init=/linuxrc console=ttyAMA0 console=ttyS0" \
    -hda disk.ext3
```

运行，依然报错：

```log
[    0.927382] uart-pl011 9000000.pl011: no DMA platform data
[    0.932510] /dev/root: Can't open blockdev
[    0.933619] VFS: Cannot open root device "/dev/sda" or unknown-block(0,0): error -6
[    0.933919] Please append a correct "root=" boot option; here are the available partitions:
[    0.934481] fe00           32768 vda 
[    0.934649]  driver: virtio_blk
[    0.934939] 1f00          131072 mtdblock0 
[    0.934963]  (driver?)
```

可以看出是root不对，应该改成vda：

```shell
#!/bin/bash

qemu-system-aarch64 \
    -nographic \
    -M virt \
    -cpu cortex-a57 \
    -smp 2 \
    -m 4G \
    -kernel /mnt/sdb/dev/linux/build/arch/arm64/boot/Image \
    -append "nokaslr root=/dev/vda init=/linuxrc console=ttyAMA0 console=ttyS0" \
    -hda disk.ext3
```

再次运行：

![image](https://github.com/ZR233/learn_rust_for_linux/blob/main/image/qemu_ok.png)

成功！


## Day 3

今天写一个驱动的`helloworld`。
首先为`kernel`添加`rust-analyzer`支持。在源码根目录执行：
```shell
make ARCH=arm64 LLVM=1 rust-analyzer
```
会在`build`目录生成`rust-project.json`，在`.vscode`的`settings.json`添加如下：
```json5
{
    "rust-analyzer.linkedProjects": [
        "./build/rust-project.json"
    ]
}
```
就可以为源码添加提示啦。
之后如教程所讲，添加`rust_helloworld`模块，编译后挂载文件系统，找个文件夹放进去，启动`qemu`，日志等级调成`info`： `dmesg -n 7`，加载驱动，成功。

## Day4

休息一天。

## Day5

开始做e1000驱动，首先尝试使用`fujita`的`kernel fork`，发现他的版本较低，用的`rust`版本为`1.66`，本着学习就学最新的原则，准备直接用官方最新版本修改使用。
`fork`了官方版本，发现`rust`只支持`x86`，对比了`rust-dev`分支，将修改支持`arm64`的部分拷贝过来，编译并运行成功。

## Day6

对比`fujita`的分支，补全官方没有的`API`。
补了一天，发现需要改动的太多，6.6 的api也有部分变动，后面的实验也不用这个项目，所以还是老实用`fujita`的吧。

## Day7

经过前两天对内核源码的捣鼓，有了些许了解，编译`kernel`，修改代码，练习3轻松完成。


## Day8

使用`busybox`需要手动添加`/etc/network/interfaces`文件：

```
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet dhcp
```

参考`e1000`驱动源码，填写练习4内容，始终无法获取网络，编译了原版`e1000`驱动测试，也不成功，看来是`qemu`配置问题。

## Day9

到处搜索资料并测试不同配置，发现问题出在没有配置本地回环地址，执行以下语句：

```shell
# 添加本地回环
ifconfig lo 127.0.0.1
# 启动网卡
ifconfig eth0 up
```

成功`ping`通。


## Day 17

riscv环境配置

busybox 编译

```shell
make  O=build_riscv ARCH=riscv64 CROSS_COMPILE=riscv64-linux-gnu-  menuconfig -j4

# setting 中打开 build static 

make  O=build_riscv ARCH=riscv64 CROSS_COMPILE=riscv64-linux-gnu- install -j4
```

linux 编译

```shell
make LLVM=1  ARCH=riscv defconfig -j4 
make LLVM=1  ARCH=riscv menuconfig -j4 
make LLVM=1  ARCH=riscv rust-analyzer
bear -- make LLVM=1  ARCH=riscv -j4 
```

## Day 18

给 rust-dev 添加riscv支持：
```makefile
# scripts/Makefile


# 22行添加，使其生成target 配置
ifdef CONFIG_RISCV
always-$(CONFIG_RUST)					+= target.json
filechk_rust_target = $< < include/config/auto.conf

$(obj)/target.json: scripts/generate_rust_target include/config/auto.conf FORCE
	$(call filechk,rust_target)
endif
```

```rust
# scripts/generate_rust_target.rs

else if cfg.has("RISCV") {
        if cfg.has("64BIT") {
            ts.push("arch", "riscv64");
            ts.push("data-layout", "e-m:e-p:64:64-i64:64-i128:128-n64-S128");
            ts.push("llvm-target", "riscv64-linux-gnu");
            ts.push("target-pointer-width", "64");
        } else {
            ts.push("arch", "riscv32");
            ts.push("data-layout", "e-m:e-p:32:32-i64:64-n32-S128");
            ts.push("llvm-target", "riscv32-linux-gnu");
            ts.push("target-pointer-width", "32");
        }
        ts.push("code-model", "medium");
        ts.push("disable-redzone", true);
        let mut features = "+m,+a".to_string();
        if cfg.has("RISCV_ISA_C") {
            features += ",+c";
        }
        ts.push("features", features);
    }

```


```Makefile
# arch/riscv/Makefile

    KBUILD_RUSTFLAGS += --target=$(objtree)/scripts/target.json

```

尝试编译，成功。
