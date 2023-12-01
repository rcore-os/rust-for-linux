---
Title: Rust for Linux驱动模块开发报告-邝劲强
Date: 2023-11
Tags:
    - Author: 邝劲强
    - Reports Repo: https://github.com/lispking/rust-for-linux/tree/main/reports
---

# Exercise 1
## 练习1: Rust for Linux 仓库源码获取，编译环境部署，及Rust内核编译。之后尝试在模拟器Qemu上运行起来。

### 以下是整个练习过程使用到的命令及相关截图结果：

因为 `Rust-for-Linux/linux` 仓库下载确实太慢，所以，这次实验使用仓库是 `fujita/linux`：

```shell
git clone git@github.com:fujita/linux.git -b rust-e1000 --depth 1 linux-rust-e1000
```

> 注：这里用了两个小技巧：
> 1. 使用 `git` 协议下载，可以非常稳定将代码下载到本地
> 2. 不需要关注历史 commit 信息，加上 `--depth 1`命令


### 0. 环境准备

#### 环境说明

* 机器配置： `CPU：M1；内存：16G；磁盘：256G`
* 主机OS：`macOS Sonoma 14.0`
* 虚拟机：`Ubuntu 23.04`

#### 环境依赖

```shell
## first Update
sudo apt-get update

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
make LLVM=1 rustavailable
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src

## Rust-for-Linux/linux 官方 bindgen 版本较新，需将命令改成 bindgen-cli
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
```

> 小技巧：若本地已经安装过 `bindgen`，在执行 `make LLVM=1 rustavailable` 会提示版本有差异，可以使用 `--force` 命令强制安装当前 `linux` 源码需要用到的版本。

### 1. 生成默认配置

```shell
make ARCH=arm64 LLVM=1 O=build defconfig
```

![make-defconfig-result](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/make-defconfig-result.png)


### 2. 支持 Rust 选项开启
```shell
make ARCH=arm64 LLVM=1 O=build menuconfig
```

#### 2.1 进入配置页面，按回车选择 `General setup`

![General-setup](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/General-setup.png)

#### 2.2 滚动到最下方，按空格选择 `Rust support`，然后，按 `ESC` 返回上一层

![make-menuconfig-result](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/make-menuconfig-result.png)

#### 2.3 再按一次 `ESC` 弹窗，按回车确认 `Yes` 即可保存配置
![Yes-confirm](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/Yes-confirm.png)

#### 2.4 配置保存结果如下图所示：

![save-config](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/save-config.png)

### 3. 编译内核

```shell
cd build && time make ARCH=arm64 LLVM=1 -j8
```

> 编译结果如下图所示，代表编译成功（整体时间花费约 `半小时`），若编译出错，最后会出现 `Error` 字眼

![compile-result](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/compile-result.png)


### 4. 进入 `Qemu` 环境验证

#### 4.1 准备 `busybox` 工具

```shell
## 下载，写作时，最新版本是 busybox-1.36.1
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
## 解压
tar xjf busybox-1.36.1.tar.bz2
## 进入目录
cd busybox-1.36.1
## 开启静态二进制，架构用 arm64
make menuconfig ARCH=arm64
```

> 进入 `busybox` 菜单选择页，在 `Settings` 按回车进入下一个页面

![busybox-setting](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/busybox-setting.png)

> 找到 `Build static binary (no shared libs)` 选项，并按空格选中后，按 `ESC` 返回上层，再按一次进入提示保存页面，回车选择 `Yes` 保存即完成静态二进制编译支持。

![busybox-setting-build-staice-binary](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise1/images/busybox-setting-build-static-binary.png)


> 编译 busybox 二进制文件，并将该执行文件 copy 到 linux 源码目录

```shell
## 编译 busybox，编译成功后，会生成 busybox 可执行文件在 busybox-1.36.1 目录下
make -j8
## 拷贝文件到 rust-for-linux 工程项目根目录
cp /tmp/busybox-1.36.1/busybox $HOME/linux/
```

#### 4.2 开启 `Rust Samples`

* 按 [2. 支持 Rust 选项开启](#2-支持-rust-选项开启) 步骤，重新进入 `menuconfig`，开启所有 `Rust Samples`

* 增加 `rust_module_parameters` 模块，并将文件名及源码内容作如下修改

```shell
cd $HOME/linux

cp samples/rust/rust_module_parameters.rs samples/rust/rust_module_parameters_builtin_default.rs
cp samples/rust/rust_module_parameters.rs samples/rust/rust_module_parameters_builtin_custom.rs
cp samples/rust/rust_module_parameters.rs samples/rust/rust_module_parameters_loadable_default.rs
cp samples/rust/rust_module_parameters.rs samples/rust/rust_module_parameters_loadable_custom.rs

sed -i 's:rust_module_parameters:rust_module_parameters_builtin_default:g'  samples/rust/rust_module_parameters_builtin_default.rs
sed -i 's:rust_module_parameters:rust_module_parameters_builtin_custom:g'   samples/rust/rust_module_parameters_builtin_custom.rs
sed -i 's:rust_module_parameters:rust_module_parameters_loadable_default:g' samples/rust/rust_module_parameters_loadable_default.rs
sed -i 's:rust_module_parameters:rust_module_parameters_loadable_custom:g'  samples/rust/rust_module_parameters_loadable_custom.rs

echo 'obj-y	+= rust_module_parameters_builtin_default.o'  >> samples/rust/Makefile
echo 'obj-y	+= rust_module_parameters_builtin_custom.o'   >> samples/rust/Makefile
echo 'obj-m	+= rust_module_parameters_loadable_default.o' >> samples/rust/Makefile
echo 'obj-m	+= rust_module_parameters_loadable_custom.o'  >> samples/rust/Makefile
```
* 按 [3. 编译内核](#3-编译内核) 命令，将 `Rust Samples` 源码编译成 `.ko` 文件

#### 4.3 生成 `qemu-initramfs.img` 镜像

```shell
sed -i 's:samples/rust/:build/samples/rust/:' .github/workflows/qemu-initramfs.desc

build/usr/gen_init_cpio .github/workflows/qemu-initramfs.desc > qemu-initramfs.img
```

#### 4.4 执行 `qemu` 命令验证结果

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
  -append ' \
    rust_module_parameters_builtin_custom.my_bool=n \
    rust_module_parameters_builtin_custom.my_i32=345543 \
    rust_module_parameters_builtin_custom.my_str=🦀mod \
    rust_module_parameters_builtin_custom.my_usize=84 \
    rust_module_parameters_builtin_custom.my_array=1,2,3 \
  ' \
  | sed 's:\r$::'
```

> 若执行结果如下所示，则代表内核准备一切就绪，然后，就可以愉快地开始编写内核驱动了。

```shell
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 6.1.0-rc1-gbb59bfc550bc-dirty (lispking@ubuntu) (Ubuntu clang version 15.0.7, Ubuntu LLD 15.0.7) #4 SMP PREEMPT Tue Nov  7 00:39:03 CST 2023
[    0.000000] random: crng init done
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x47fb0a00-0x47fb2fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000] cma: Reserved 32 MiB at 0x0000000045c00000
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] psci: SMC Calling Convention v1.0
[    0.000000] percpu: Embedded 20 pages/cpu s44840 r8192 d28888 u81920
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: Spectre-v3a
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] CPU features: detected: Spectre-BHB
[    0.000000] CPU features: kernel page table isolation forced ON by KASLR
[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)
[    0.000000] CPU features: detected: ARM erratum 1742098
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] alternatives: applying boot alternatives
[    0.000000] Fallback order for Node 0: 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32256
[    0.000000] Policy zone: DMA
[    0.000000] Kernel command line:  \
[    0.000000]     rust_module_parameters_builtin_custom.my_bool=n \
[    0.000000]     rust_module_parameters_builtin_custom.my_i32=345543 \
[    0.000000]     rust_module_parameters_builtin_custom.my_str=🦀mod \
[    0.000000]     rust_module_parameters_builtin_custom.my_usize=84 \
[    0.000000]     rust_module_parameters_builtin_custom.my_array=1,2,3 \
[    0.000000]
[    0.000000] Unknown kernel command line parameters "\ \ \ \ \ \", will be passed to user space.
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] Memory: 58068K/131072K available (16640K kernel code, 3724K rwdata, 9340K rodata, 1920K init, 610K bss, 40236K reserved, 32768K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU event tracing is enabled.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] 	Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: gic_handle_irq
[    0.000000] GICv2m: range[mem 0x08020000-0x08020fff], SPI[80:143]
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0x1ffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000034] sched_clock: 57 bits at 63MHz, resolution 16ns, wraps every 4398046511096ns
[    0.003104] Console: colour dummy device 80x25
[    0.006498] printk: console [tty0] enabled
[    0.007580] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.007696] pid_max: default: 32768 minimum: 301
[    0.008201] LSM: Security Framework initializing
[    0.009621] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.009667] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.027385] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.030707] cblist_init_generic: Setting adjustable number of callback queues.
[    0.030813] cblist_init_generic: Setting shift to 1 and lim to 1.
[    0.031032] cblist_init_generic: Setting shift to 1 and lim to 1.
[    0.031887] rcu: Hierarchical SRCU implementation.
[    0.031927] rcu: 	Max phase no-delay instances is 1000.
[    0.034754] EFI services will not be available.
[    0.035301] smp: Bringing up secondary CPUs ...
[    0.038000] Detected PIPT I-cache on CPU1
[    0.038422] cacheinfo: Unable to detect cache hierarchy for CPU 1
[    0.038656] CPU1: Booted secondary processor 0x0000000001 [0x410fd083]
[    0.040629] smp: Brought up 1 node, 2 CPUs
[    0.041139] SMP: Total of 2 processors activated.
[    0.041197] CPU features: detected: 32-bit EL0 Support
[    0.041224] CPU features: detected: 32-bit EL1 Support
[    0.041273] CPU features: detected: CRC32 instructions
[    0.042832] CPU: All CPU(s) started at EL1
[    0.042955] alternatives: applying system-wide alternatives
[    0.053178] devtmpfs: initialized
[    0.060754] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.060882] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.063074] pinctrl core: initialized pinctrl subsystem
[    0.069593] DMI not present or invalid.
[    0.097328] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.103269] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.103577] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.103741] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.103955] audit: initializing netlink subsys (disabled)
[    0.104990] audit: type=2000 audit(0.064:1): state=initialized audit_enabled=0 res=1
[    0.107444] thermal_sys: Registered thermal governor 'step_wise'
[    0.107475] thermal_sys: Registered thermal governor 'power_allocator'
[    0.107695] cpuidle: using governor menu
[    0.108455] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.108858] ASID allocator initialised with 32768 entries
[    0.111764] Serial: AMBA PL011 UART driver
[    0.133623] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 13, base_baud = 0) is a PL011 rev1
[    0.138161] printk: console [ttyAMA0] enabled
[    0.144714] KASLR enabled
[    0.172047] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.172209] HugeTLB: 16380 KiB vmemmap can be freed for a 1.00 GiB page
[    0.172317] HugeTLB: registered 32.0 MiB page size, pre-allocated 0 pages
[    0.172405] HugeTLB: 508 KiB vmemmap can be freed for a 32.0 MiB page
[    0.172490] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
[    0.172588] HugeTLB: 28 KiB vmemmap can be freed for a 2.00 MiB page
[    0.172694] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages
[    0.172794] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page
[    0.178560] ACPI: Interpreter disabled.
[    0.182045] iommu: Default domain type: Translated
[    0.182153] iommu: DMA domain TLB invalidation policy: strict mode
[    0.183245] SCSI subsystem initialized
[    0.184691] usbcore: registered new interface driver usbfs
[    0.184938] usbcore: registered new interface driver hub
[    0.185193] usbcore: registered new device driver usb
[    0.186663] pps_core: LinuxPPS API ver. 1 registered
[    0.186741] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.186899] PTP clock support registered
[    0.187237] EDAC MC: Ver: 3.0.0
[    0.191035] FPGA manager framework
[    0.191539] Advanced Linux Sound Architecture Driver Initialized.
[    0.197068] vgaarb: loaded
[    0.201654] clocksource: Switched to clocksource arch_sys_counter
[    0.203078] VFS: Disk quotas dquot_6.6.0
[    0.203335] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.204321] pnp: PnP ACPI: disabled
[    0.233969] NET: Registered PF_INET protocol family
[    0.235788] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.240230] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.240647] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.240810] TCP established hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.241319] TCP bind hash table entries: 1024 (order: 3, 32768 bytes, linear)
[    0.241642] TCP: Hash tables configured (established 1024 bind 1024)
[    0.242672] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.243137] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.244082] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.246729] RPC: Registered named UNIX socket transport module.
[    0.246978] RPC: Registered udp transport module.
[    0.247087] RPC: Registered tcp transport module.
[    0.247207] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.247457] PCI: CLS 0 bytes, default 64
[    0.250194] Unpacking initramfs...
[    0.252089] hw perfevents: enabled with armv8_pmuv3 PMU driver, 7 counters available
[    0.252719] kvm [1]: HYP mode not available
[    0.255075] Initialise system trusted keyrings
[    0.256216] workingset: timestamp_bits=42 max_order=15 bucket_order=0
[    0.264587] Freeing initrd memory: 4048K
[    0.264813] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.266781] NFS: Registering the id_resolver key type
[    0.267102] Key type id_resolver registered
[    0.267177] Key type id_legacy registered
[    0.267604] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.267759] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    0.268362] 9p: Installing v9fs 9p2000 file system support
[    0.296720] Key type asymmetric registered
[    0.296866] Asymmetric key parser 'x509' registered
[    0.297224] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 245)
[    0.297383] io scheduler mq-deadline registered
[    0.297517] io scheduler kyber registered
[    0.313985] pl061_gpio 9030000.pl061: PL061 GPIO chip registered
[    0.335794] pci-host-generic 4010000000.pcie: host bridge /pcie@10000000 ranges:
[    0.336496] pci-host-generic 4010000000.pcie:       IO 0x003eff0000..0x003effffff -> 0x0000000000
[    0.336995] pci-host-generic 4010000000.pcie:      MEM 0x0010000000..0x003efeffff -> 0x0010000000
[    0.337139] pci-host-generic 4010000000.pcie:      MEM 0x8000000000..0xffffffffff -> 0x8000000000
[    0.337729] pci-host-generic 4010000000.pcie: Memory resource size exceeds max for 32 bits
[    0.338221] pci-host-generic 4010000000.pcie: ECAM at [mem 0x4010000000-0x401fffffff] for [bus 00-ff]
[    0.339179] pci-host-generic 4010000000.pcie: PCI host bridge to bus 0000:00
[    0.339454] pci_bus 0000:00: root bus resource [bus 00-ff]
[    0.339581] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.339685] pci_bus 0000:00: root bus resource [mem 0x10000000-0x3efeffff]
[    0.339829] pci_bus 0000:00: root bus resource [mem 0x8000000000-0xffffffffff]
[    0.341081] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000
[    0.343668] pci 0000:00:01.0: [1af4:1000] type 00 class 0x020000
[    0.343989] pci 0000:00:01.0: reg 0x10: [io  0x0000-0x001f]
[    0.344113] pci 0000:00:01.0: reg 0x14: [mem 0x00000000-0x00000fff]
[    0.344331] pci 0000:00:01.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[    0.344512] pci 0000:00:01.0: reg 0x30: [mem 0x00000000-0x0007ffff pref]
[    0.346415] pci 0000:00:01.0: BAR 6: assigned [mem 0x10000000-0x1007ffff pref]
[    0.346725] pci 0000:00:01.0: BAR 4: assigned [mem 0x8000000000-0x8000003fff 64bit pref]
[    0.346953] pci 0000:00:01.0: BAR 1: assigned [mem 0x10080000-0x10080fff]
[    0.347074] pci 0000:00:01.0: BAR 0: assigned [io  0x1000-0x101f]
[    0.351329] EINJ: ACPI disabled.
[    0.374286] virtio-pci 0000:00:01.0: enabling device (0000 -> 0003)
[    0.385972] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    0.390186] SuperH (H)SCI(F) driver initialized
[    0.391020] msm_serial: driver initialized
[    0.393221] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.416646] loop: module loaded
[    0.418396] megasas: 07.719.03.00-rc1
[    0.422165] physmap-flash 0.flash: physmap platform flash device: [mem 0x00000000-0x03ffffff]
[    0.423432] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    0.423866] Intel/Sharp Extended Query Table at 0x0031
[    0.424794] Using buffer write method
[    0.425221] physmap-flash 0.flash: physmap platform flash device: [mem 0x04000000-0x07ffffff]
[    0.426214] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    0.426376] Intel/Sharp Extended Query Table at 0x0031
[    0.426803] Using buffer write method
[    0.426954] Concatenating MTD devices:
[    0.427018] (0): "0.flash"
[    0.427070] (1): "0.flash"
[    0.427111] into device "0.flash"
[    0.447101] tun: Universal TUN/TAP device driver, 1.6
[    0.455641] thunder_xcv, ver 1.0
[    0.455774] thunder_bgx, ver 1.0
[    0.455875] nicpf, ver 1.0
[    0.457608] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version
[    0.457692] hns3: Copyright (c) 2017 Huawei Corporation.
[    0.458149] hclge is initializing
[    0.458287] e1000: Intel(R) PRO/1000 Network Driver
[    0.458358] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    0.458499] e1000e: Intel(R) PRO/1000 Network Driver
[    0.458558] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    0.458704] igb: Intel(R) Gigabit Ethernet Network Driver
[    0.458766] igb: Copyright (c) 2007-2014 Intel Corporation.
[    0.459007] igbvf: Intel(R) Gigabit Virtual Function Network Driver
[    0.459083] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    0.459626] sky2: driver version 1.30
[    0.461268] VFIO - User Level meta-driver version: 0.3
[    0.466328] usbcore: registered new interface driver usb-storage
[    0.471491] rtc-pl031 9010000.pl031: registered as rtc0
[    0.472031] rtc-pl031 9010000.pl031: setting system clock to 2023-11-07T06:02:08 UTC (1699336928)
[    0.473346] i2c_dev: i2c /dev entries driver
[    0.482225] sdhci: Secure Digital Host Controller Interface driver
[    0.482316] sdhci: Copyright(c) Pierre Ossman
[    0.483338] Synopsys Designware Multimedia Card Interface Driver
[    0.484559] sdhci-pltfm: SDHCI platform and OF driver helper
[    0.487468] ledtrig-cpu: registered to indicate activity on CPUs
[    0.490437] usbcore: registered new interface driver usbhid
[    0.490514] usbhid: USB HID core driver
[    0.499497] NET: Registered PF_PACKET protocol family
[    0.500241] 9pnet: Installing 9P2000 support
[    0.500450] Key type dns_resolver registered
[    0.501267] registered taskstats version 1
[    0.501467] Loading compiled-in X.509 certificates
[    0.520843] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    0.543124] ALSA device list:
[    0.543312]   No soundcards found.
[    0.546543] uart-pl011 9000000.pl011: no DMA platform data
[    0.612888] Freeing unused kernel memory: 1920K
[    0.614508] Run /init as init process
[    0.666661] rust_minimal: Rust minimal sample (init)
[    0.666979] rust_minimal: Am I built-in? false
[    0.675964] rust_minimal: My numbers are [72, 108, 200]
[    0.676445] rust_minimal: Rust minimal sample (exit)
[    0.720870] rust_print: Rust printing macros sample (init)
[    0.721035] rust_print: Emergency message (level 0) without args
[    0.721234] rust_print: Alert message (level 1) without args
[    0.721483] rust_print: Critical message (level 2) without args
[    0.721705] rust_print: Error message (level 3) without args
[    0.721774] rust_print: Warning message (level 4) without args
[    0.721836] rust_print: Notice message (level 5) without args
[    0.721913] rust_print: Info message (level 6) without args
[    0.721982] rust_print: A line that is continued without args
[    0.722114] rust_print: Emergency message (level 0) with args
[    0.722220] rust_print: Alert message (level 1) with args
[    0.722285] rust_print: Critical message (level 2) with args
[    0.722362] rust_print: Error message (level 3) with args
[    0.722426] rust_print: Warning message (level 4) with args
[    0.722496] rust_print: Notice message (level 5) with args
[    0.722589] rust_print: Info message (level 6) with args
[    0.722715] rust_print: A line that is continued with args
[    0.727927] rust_print: Rust printing macros sample (exit)
[    0.749354] rust_module_parameters: Rust module parameters sample (init)
[    0.749642] rust_module_parameters: Parameters:
[    0.750058] rust_module_parameters:   my_bool:    true
[    0.750170] rust_module_parameters:   my_i32:     42
[    0.750430] rust_module_parameters:   my_str:     default str val
[    0.750545] rust_module_parameters:   my_usize:   42
[    0.750700] rust_module_parameters:   my_array:   [0, 1]
[    0.755527] rust_module_parameters: Rust module parameters sample (exit)
[    0.780268] rust_sync: Rust synchronisation primitives sample (init)
[    0.780446] rust_sync: Value: 10
[    0.780799] rust_sync: Value: 10
[    0.785505] rust_sync: Rust synchronisation primitives sample (exit)
[    0.813859] rust_chrdev: Rust character device sample (init)
[    0.819166] rust_chrdev: Rust character device sample (exit)
[    0.841260] rust_miscdev: Rust miscellaneous device sample (init)
[    0.847056] rust_miscdev: Rust miscellaneous device sample (exit)
[    0.872530] rust_stack_probing: Rust stack probing sample (init)
[    0.872732] rust_stack_probing: Large array has length: 514
[    0.877872] rust_stack_probing: Rust stack probing sample (exit)
[    0.906001] rust_semaphore: Rust semaphore sample (init)
[    0.911501] rust_semaphore: Rust semaphore sample (exit)
[    0.936540] rust_semaphore_c: Rust semaphore sample (in C, for comparison) (init)
[    0.941912] rust_semaphore_c: Rust semaphore sample (in C, for comparison) (exit)
[    0.964929] rust_selftests: Rust self tests (init)
[    0.965048] rust_selftests: test_example passed!
[    0.965115] rust_selftests: 1 tests run, 1 passed, 0 failed, 0 hit errors
[    0.965453] rust_selftests: All tests passed. Congratulations!
[    0.971363] rust_selftests: Rust self tests (exit)
[    0.997060] rust_module_parameters_loadable_default: Rust module parameters sample (init)
[    0.997242] rust_module_parameters_loadable_default: Parameters:
[    0.997390] rust_module_parameters_loadable_default:   my_bool:    true
[    0.997531] rust_module_parameters_loadable_default:   my_i32:     42
[    0.997992] rust_module_parameters_loadable_default:   my_str:     default str val
[    0.998271] rust_module_parameters_loadable_default:   my_usize:   42
[    0.998512] rust_module_parameters_loadable_default:   my_array:   [0, 1]
[    1.007882] rust_module_parameters_loadable_custom: Rust module parameters sample (init)
[    1.008054] rust_module_parameters_loadable_custom: Parameters:
[    1.008180] rust_module_parameters_loadable_custom:   my_bool:    false
[    1.008445] rust_module_parameters_loadable_custom:   my_i32:     345543
[    1.008939] rust_module_parameters_loadable_custom:   my_str:     🦀mod
[    1.009066] rust_module_parameters_loadable_custom:   my_usize:   84
[    1.009352] rust_module_parameters_loadable_custom:   my_array:   [1, 2, 3]
[    1.016364] rust_module_parameters_loadable_default: Rust module parameters sample (exit)
[    1.035957] rust_module_parameters_loadable_custom: Rust module parameters sample (exit)
[    1.068096] Flash device refused suspend due to active operation (state 20)
[    1.068362] Flash device refused suspend due to active operation (state 20)
[    1.068791] reboot: Restarting system
```

# Exercise 2

## 练习2: 自定义编写Rust内核驱动模块

We will only do in-tree compilation. In this part, we need to build a minimal kernel module

* 编写 `rust_helloworld` 模块

```shell
## 进入练习1所在源码目录
cd $HOME/linux

## 添加 RustHelloWorld 模块源码
cat <<EOF > samples/rust/rust_helloworld.rs
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
EOF
```

* 添加编译相关配置

```shell
## 在 samples/rust/Makefile 文件添加以下内容：
obj-$(CONFIG_SAMPLE_RUST_HELLOWORLD)        += rust_helloworld.o

## 在 samples/rust/Kconfig 文件添加以下内容：
config SAMPLE_RUST_HELLOWORLD
  tristate "Print Hello World in Rust"
  help
    This option builds the Rust HelloWorld module sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_helloworld.
      
    If unsure, say N.

## 在练习1基础上，添加 rust_helloworld 模块配置
make ARCH=arm64 LLVM=1 O=build menuconfig
```

![hello-world-config](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise2/images/hello-world-config.png)

* 编译 `rust_helloworld` 模块

```shell
## 将 rust_helloworld 模块编译，编译成功后，会生成 build/samples/rust/rust_helloworld.ko 文件
cd build && time make ARCH=arm64 LLVM=1 -j8
```

* 修改 `qemu-init.sh` 脚本，增加 `rust_helloworld` 模块

```shell
busybox insmod rust_helloworld.ko
busybox  rmmod rust_helloworld.ko
```

* 修改镜像描述文件 `qemu-initramfs.desc`，增加 `rust_helloworld` 模块

```shell
file    /rust_helloworld.ko         samples/rust/rust_helloworld.ko         0755 0 0
```

* 启动 `qemu` 验证

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
  -append ' \
    rust_module_parameters_builtin_custom.my_bool=n \
    rust_module_parameters_builtin_custom.my_i32=345543 \
    rust_module_parameters_builtin_custom.my_str=🦀mod \
    rust_module_parameters_builtin_custom.my_usize=84 \
    rust_module_parameters_builtin_custom.my_array=1,2,3 \
  ' \
  | sed 's:\r$::'
```

* 验证结果显示如下面红色框，则表示`rust_helloworld` 模块加载成功。

![hello-world-result](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise2/images/hello-world-result.png)

* 代码已上传至：https://github.com/lispking/rust-for-linux-e1000/commit/fe8497d45e5cf719a4c1816c4183c80a55816899

## 基于Qemu模拟器上的e1000网卡驱动框架，填充驱动初始化函数

1. 由于在练习1里面是用的.github/workflow里的脚本生成的镜像，那个镜像的问题是不会停留在内核里，所以需要重新做一个

```shell
cd busybox-1.36.1

# 上一次练习没有执行这一步
make install
```

2. 由于实验用的是 `fujita/linux` 源码，需要将自带的 `e1000` 驱动先去掉，再重新编译内核

3. 启动脚本 `qemu-aarch64-test.sh` 改成下面：

```shell
#!/bin/bash

BUSYBOX=../busybox-1.36.1
INITRD=${PWD}/initramfs.cpio.gz
BUSYBOX_INSTALL_DIR=$BUSYBOX/_install
MODULE_DIR=/lib/modules

cat <<EOF > $BUSYBOX_INSTALL_DIR/init
#!/bin/busybox sh

/bin/busybox mkdir -p /proc && /bin/busybox mount -t proc none /proc

/bin/busybox echo -e "\033[33m[$(date)] Hello, Welcome to Rust for Linux! \033[0m"

/bin/busybox ls $MODULE_DIR
/bin/busybox insmod $MODULE_DIR/e1000_for_linux.ko

/bin/busybox ip addr add 127.0.0.1/32 dev lo
/bin/busybox ip link set lo up

/bin/busybox ip addr add 192.168.100.223/24 dev eth0
/bin/busybox ip link set eth0 up

export 'PS1=(kernel) >'
/bin/busybox sh
EOF

# 参考这里: https://www.jianshu.com/p/9b68e9ea5849
# Set host-only network
if [ "$1" == "init-host-only" ]; then
  sudo ip link add br0 type bridge
  sudo ip addr add 192.168.100.50/24 brd 192.168.100.255 dev br0
  sudo ip tuntap add mode tap user $(whoami)
  ip tuntap show
  sudo ip link set tap0 master br0
  sudo ip link set dev br0 up
  sudo ip link set dev tap0 up
fi

chmod +x $BUSYBOX_INSTALL_DIR/init

mkdir -p $BUSYBOX_INSTALL_DIR/$MODULE_DIR/

## 网卡驱动 ko 文件
cp ../rust-e1000-driver/src/e1000_for_linux.ko $BUSYBOX_INSTALL_DIR/$MODULE_DIR/

cd $BUSYBOX_INSTALL_DIR && find . -print0 | cpio --null -ov --format=newc | gzip -9 > ${INITRD} && cd -

qemu-system-aarch64 \
  -kernel ./build/arch/arm64/boot/Image.gz \
  -initrd ${INITRD} \
  -M virt \
  -cpu cortex-a72 \
  -smp 2 \
  -m 128M \
  -nographic \
  -netdev tap,ifname=tap0,id=tap0,script=no,downscript=no -device e1000,netdev=tap0 \
  -append 'init=/init console=ttyAMA0'
```

4. 左边窗口执行 `./qemu-aarch64-test.sh`，网卡驱动和 `ip` 地址在 `init` 脚本里已实现自动加载，待 `qemu` 启动后，在右边窗口执行 `ping` 命令，运行成功，结果将如下所示：

![ping-pong](https://github.com/lispking/rust-for-linux/blob/main/reports/exercise3/images/ping-pong.png)
