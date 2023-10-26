# 自定义一个linux内核函数, 并在rust模块中调用它.

# step1
在linux include目录下创建一个头文件xx.h, 在里面写好自定义的一个函数, 打印一行log出来. 

# step2
在linux rust/bindings/bindings_helper.h中把step1中的头文件引用进来. 在rust/helpers.c中参考其中已有例子,生成rust_helper_为前缀的函数.

# step3
在之前的helloworld模块中引入kernel::bindings::*; 通过bindings::函数名方式调用.


