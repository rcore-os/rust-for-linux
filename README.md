# Rust for Linux 驱动模块开发

## 课题目标
学生选修该课题，可以学习到如何一步一步地开发一个Rust for Linux内核驱动；逐步积累并形成学习开发文档；进而有机会为rcore-os Rust操作系统社区以及Linux社区的发展，通过编写Rust驱动等方式贡献一份自己的力量。

## 相关资料参考
Rust for Linux社区仓库: https://github.com/Rust-for-Linux

Rust e1000网卡驱动: https://github.com/fujita/linux.git

Rust实现的VirtIO驱动: https://github.com/rcore-os/virtio-drivers.git

RISCV物理开发板平台:华山派: https://github.com/sophgo/sophpi-huashan.git


## Rust for Linux 设备驱动实现案例

### Linux上的Rust驱动程序

* Western Digital 首席工程师 Andreas Hindborg 在2022 Linux Plumbers Summit上
展示了用户可以使用 Rust 编写一流的驱动程序，即适用于 Linux 的 SSD NVM-Express (NVMe) 驱动程序。 <br>
Linux (PCI) NVMe driver in Rust <br>
https://lpc.events/event/16/contributions/1180/attachments/1017/1961/deck.pdf
https://github.com/metaspace/linux/commits/nvme

* Asahi Lina Linux组织开发的Apple's M1 GPU驱动 <br>
https://github.com/AsahiLinux/linux/commits/gpu/rust-wip

* 对async异步内核编程的支持，当前状态已经用于允许异步网络TCP连接支持； <br>
https://github.com/Rust-for-Linux/linux/commit/08358e5a955f662daf154a89f3ba43a3f18228a2

* Google Android 团队正试验移植Rust Binder IPC 驱动，其它 Android 厂商也表示有兴趣开发 Rust 驱动 <br>
https://github.com/Rust-for-Linux/linux/commit/d8dcaf80c9535056ea541d8dd876edad52c538a9

* 9p fs server 方便地从虚拟主机与Host机进行文件系统访问 <br>
https://kangrejos.com/Async%20Rust%20and%209p%20server.pdf
https://github.com/wedsonaf/linux/commits/9p

* Samsung - Null块驱动：Null Block Driver <br>
https://github.com/metaspace/linux/tree/null_block-RFC

* Cisco - 容器文件系统： PuzzleFS <br>
https://github.com/project-machine/puzzlefs

* Microsoft TarFS <br>
https://github.com/wedsonaf/linux/commits/vfs

* Collabora - Video Codec <br>
https://lore.kernel.org/rust-for-linux/20230406215615.122099-1-daniel.almeida@collabora.com/
