# 基于Qemu模拟器上的e1000网卡驱动框架，填充驱动初始化函数
获取e1000-driver代码 https://github.com/yuoo655/e1000-driver 以及相应的linux仓库 https://github.com/fujita/linux/tree/rust-e1000

完成Exercise3 Checkpoint 1-5

# 自定义一个linux内核函数, 并在rust模块中调用它.

# step1
在linux include目录下创建一个头文件xx.h, 在里面写好自定义的一个函数, 打印一行log出来. 

# step2
在linux rust/bindings/bindings_helper.h中把step1中的头文件引用进来. 在rust/helpers.c中参考其中已有例子,生成rust_helper_为前缀的函数.

# step3
在Rust网卡驱动模块中模块中引入kernel::bindings::*; 通过bindings::函数名方式调用.


