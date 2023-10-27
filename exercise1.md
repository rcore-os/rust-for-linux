# Exercise 1
## 练习1: Rust for Linux 仓库源码获取，编译环境部署，及Rust内核编译。

1. Get source code 
we have collected all the source code for you in the homework repository(as mentioned before). However, you can also get source code from Rust for Linux official repository:
```
git clone https://github.com/Rust-for-Linux/linux
```
and the rust version source code by XLY from the following repository:
```
git clone https://github.com/fujita/linux.git -b rust-e1000
```
we have merged the two repositories and fixed the bugs in building. But if you are interested in how it works, you can try to merge it and fix the conflicts. Enjoy it!

2. Preparation
First you need to install some packages(if the warning or error tell you need more packages, just intall them all):
```
sudo apt-get -y install \
  binutils build-essential libtool texinfo \
  gzip zip unzip patchutils curl git \
  make cmake ninja-build automake bison flex gperf \
  grep sed gawk bc \
  zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev libelf-dev libssl-dev
```

Besides a Rust environment is necessary, but I think you have done this in stage1.
And install the LLVM:
```
apt-get install clang-format clang-tidy clang-tools clang clangd libc++-dev libc++1 libc++abi-dev libc++abi1 libclang-dev libclang1 liblldb-dev libllvm-ocaml-dev libomp-dev libomp5 lld lldb llvm-dev llvm-runtime llvm python3-clang
```

Then try to make the kernel support the Rust language:
```
cd linux
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
sudo apt install clang llvm
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
rustup component add rustfmt
rustup component add clippy
make LLVM=1 rustavailable
```
 
3. Build the kernel tree and the image
As we introduced in our lesson, you have to
```
make LLVM=1 menuconfig
#set the following config to yes
General setup
        ---> [*] Rust support
make LLVM=1 -j$(nproc)
```
After that, under your linux folder you can find a file named vmlinux, which means your building is success. Congratulations!

Remember, when you building the kernel, some errors might happen, which is very normal and how to fix the bug or just ignore it, is a part of your homework.
For example, some modules might show an error called "modpost \"xxx\" [xxx.ko] underfined!" on my assistant's environment, but that's just an error of modules and have  no affection on the kernel image, so just ignore it. 
If you have any errors during the process of building the kernel tree, we do not restrict you to use any method to solve the problem, including using google, querying stackoverflow, asking chatgpt, or directly consulting the teaching assistant, etc.
Testcase and Grade
This part accounting for 60% of the score.

No testcase.
We have no way to test whether you have made a correct environment and built your kernel tree and image successfully on your computer, so you have to submit a report and show the  runtime screenshot in your report.
If errors occur during configuration, we want you to be able to document them. If it doesn't work out in any way, you can get 60 points for reporting your mistake.
