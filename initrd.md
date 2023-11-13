使用 busybox 制作内存文件系统 initramfs：

1.  下载解压 busybox 并配置环境变量

    ```shell
    wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2
    tar -xf busybox-1.35.0.tar.bz2
    cd busybox-1.35.0
    # 配置环境变量
    export ARCH=arm64
    export CROSS_COMPILE=aarch64-linux-gnu-
    ```

2.  配置编译内核的参数

    ```shell
    # busybox-1.35.0目录下
    make menuconfig
    # 修改配置，选中如下项目，静态编译
    # Settings -> Build Options -> [*] Build static binary (no share libs)


3.  编译

    ```shell
    make -j `nproc`
    ```

4.  安装

    安装前 busybox-1.35.0 目录下的文件如下图所示



    输入如下命令

    ```shell
    make install
    ```


    安装后目录下生成了\_install 目录

5.  在\_install 目录下创建后续所需的文件和目录

    ```shell
    cd _install
    mkdir proc sys dev tmp
    touch init
    chmod +x init
    ```

6.  用任意的文本编辑器编辑 init 文件内容如下

    ```shell
    #!/bin/sh

    # 挂载一些必要的文件系统
    mount -t proc none /proc
    mount -t sysfs none /sys
    mount -t tmpfs none /tmp
    mount -t devtmpfs none /dev


    # 停留在控制台
    exec /bin/sh
    ```

7.  用 busybox 制作 initramfs 文件

    ```shell
    # _install目录
    find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
    ```

    执行成功后可在 busybox-1.35.0 目录下找到 initramfs.cpio.gz 文件

8.  进入 qemu 目录下，执行如下命令

    ```shell
    qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel (your Image path) -initrd (your initramfs.cpio.gz path) -nographic -append "init=/init console=ttyAMA0"
    ```

    其中`(your Image path)`为上一个任务中最后的`Image`镜像文件所在的目录，`(your initramfs.cpio.gz)`为步骤 7 中执行成功后得到的 initramfs.cpio.gz 的目录，例如

    ```shell
    qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel /home/jun/maodou/linux/arch/arm64/boot/Image -initrd /home/jun/maodou/busybox-1.35.0/initramfs.cpio.gz -nographic -append "init=/init console=ttyAMA0"
    ```