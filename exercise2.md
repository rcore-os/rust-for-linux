# Exercise 2

## 练习2: 自定义编写Rust内核驱动模块

We will only do in-tree compilation. In this part, we need to build a minimal kernel module

1. Goto samples/rust directory in Linux folder
2. Add a rust_helloworld.rs in this directory
3. Add these content to `rust_helloworld.rs`

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
4. Edit Makefile and Kconfig within samples/rust directory

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

5. run make LLVM=1 menuconfig
6. You can enter `/` and search for the location of the module you want to find. Now make your module compile within kernel binary.
```
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <*>Print Helloworld in Rust (NEW)
```
Remember that in the previous option you need to press Y, in the last option you need to press M.
Then exit menuconfig and save your change.

7. Compile kernel again make LLVM=1 and you should see no Error if your previous steps are right. 
 
Testcase and Grade
This part accounting for 10% of the score.

If everything is ok, you can see a  rust_helloworld.ko in the samples/rust directory.
Then copy it to the src_e1000/rootfs and then run build_image.sh

### A Small TTY Driver from <<Linux Device Driver 3rd Edition>>


After you enter the Linux shell, use 'ls' command,  you can find the rust_helloworld.ko.
Then use insmod to start this module.
Finally, you can see the "Hello World from Rust module"
