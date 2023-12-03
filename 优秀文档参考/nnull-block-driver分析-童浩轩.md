---
Title: Rust for Linux驱动模块开发报告-童浩轩
Date: 2023-11
Tags:
    - Author: 童浩轩
    - Reports Repo: https://github.com/Tunghohin/rust-for-linux/blob/main/reports/final_report.md
---

# null block driver 分析

项目源码地址

> https://github.com/metaspace/linux/tree/null_block-RFC

## 什么是 null block driver？

"Null block driver" 通常是指一个虚拟的块设备驱动程序，它并不代表任何真实的硬件设备，而是用于模拟块设备的行为。在 linux 中，C 版本的 null block driver 在 drivers/block/null_blk 中。

linux 系统中，块设备系统的结构大概可以看成

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/multi-queue2-block layer.png">

进程操作 filesystem，filesystem

操作系统为每个进程分配了一个 staging queue，而 hardware dispatch queues 的数量取决于硬件

## 代码分析

#### struct blk_mq_tag_set

struct blk_mq_tag_set 是 Linux 内核中用于表示块多队列（blk-mq）层中队列标签集合的数据结构。在 blk-mq 中，队列标签（tag）用于标识和跟踪块设备上发出的 I/O 请求。

C 版本 struct blk_mq_tag_set

```c
struct blk_mq_tag_set {
    struct blk_mq_ops *ops;      // 操作函数指针
    unsigned int flags;          // 标志位
    unsigned int nr_hw_queues;    // 硬件队列数量
    unsigned int queue_depth;     // 每个队列的深度（即队列标签的数量） */
    struct request_queue *queue;  // 关联的请求队列
    void *driver_data;            // 驱动程序私有数据指针
    ...
};
```

在 rust 中，一个实用的技巧是，将 C 结构体指针用 rust 的 struct wrap 一层

```rust
pub struct TagSet<T: Operations> {
    inner: UnsafeCell<bindings::blk_mq_tag_set>,
    _p: PhantomData<T>,
}
```

其中需要注意的点是，特征约束 T 在结构体中没有使用，我们只想让其发挥约束的作用，如果强行创建一个元素占用显然是不优的。但是使用 PhantomData 可以在类型系统中表示这个类型的存在，总的来说，PhantomData 是 Rust 中用于表达类型系统中一些约束、关系或者存在性的一种方式，它在类型层面上提供了一些信息，而不引入实际的运行时开销。

注意到，blk_mq_tag_set 中，有个 ops 结构体，里面存了一系列函数指针

<img src="https://github.com/Tunghohin/rust-for-linux/blob/main/reports/./imgs/Screenshot from 2023-12-03 11-52-20.png">

在 rust 中，我们要怎样实现呢？

/kernel/rust/macro/vtable.rs 中，提供了一个 vtable 宏，我们可以通过 table 宏，将 rust 代码转换成 C 代码。
一个被标注了 #[vtable] attribute 的 trait，他的实现不应该在 rust 端被调用。

使用方法如下（以 commit_rqs 为例）

```rust
#[macro::vtable]
pub trait Operations: Sized {
    ... //以commit_rqs为例
    type HwData: ForeignOwnable;

    ...

    fn commit_rqs(
        hw_data: <Self::HwData as ForeignOwnable>::Borrowed<'_>,
        queue_data: <Self::QueueData as ForeignOwnable>::Borrowed<'_>,
    );
    ...
}

pub(crate) struct OperationsVtable<T: Operations>(PhantomData<T>);

impl<T: Operations> OperationsVtable<T> {
    ...
    unsafe extern "C" fn commit_rqs_callback(hctx: *mut bindings::blk_mq_hw_ctx) {
        let hw_data = unsafe { T::HwData::borrow((*hctx).driver_data) };

        // SAFETY: `hctx` is valid as required by this function.
        let queue_data = unsafe { (*(*hctx).queue).queuedata };

        // SAFETY: `queue.queuedata` was created by `GenDisk::try_new()` with a
        // call to `ForeignOwnable::into_pointer()` to create `queuedata`.
        // `ForeignOwnable::from_foreign()` is only called when the tagset is
        // dropped, which happens after we are dropped.
        let queue_data = unsafe { T::QueueData::borrow(queue_data) };
        T::commit_rqs(hw_data, queue_data)
    }
    ...

    const VTABLE: bindings::blk_mq_ops = bindings::blk_mq_ops {
        ...
        commit_rqs: Some(Self::commit_rqs_callback),
        ...
    };

    pub(crate) const unsafe fn build() -> &'static bindings::blk_mq_ops {
        &Self::VTABLE
    }
}
```

其中 types::ForeignOwnable 表示可用于 C 于 rust 之间的资源所有权转移

### bio

在 Linux 内核中，struct bio 是用于表示块 I/O 操作的结构体。这个结构体包含了有关块 I/O 操作的重要信息，比如要执行的操作、数据缓冲区的位置等。

struct bio 通常涉及到一个或多个 struct bio_vec，它们描述了块 I/O 操作的数据块的位置和大小。struct bio_vec 中包含了指向数据缓冲区的指针、偏移量和长度。

```c
struct bio {
    struct bio *bi_next;         // 链表中的下一个 bio 结构体
    struct block_device *bi_bdev; // 目标块设备
    unsigned long bi_flags;       // 标志位，用于描述 I/O 操作的状态
    bio_flags_t bi_rw;            // 读写标志位，指示是读操作还是写操作
    unsigned short bi_vcnt;       // 向量个数，用于描述 bio 中的数据块数量
    unsigned short bi_idx;        // 当前处理的数据块索引
    unsigned int bi_size;         // 数据块大小
    unsigned int bi_iter.bi_sector; // 起始扇区号
    struct bio_vec bi_io_vec;     // 包含数据缓冲区的信息
    struct bio_set *bi_pool;      // 内存池指针
    ...
};
```

在 rust 里，依然采用指针的形式对其进行封装，但是值得注意的是，采用了 NonNull 指针，而非裸指针

```rust
pub struct Bio<'a>(
    NonNull<crate::bindings::bio>,
    core::marker::PhantomData<&'a ()>,
);
```

NonNull 通常与裸指针一起使用，把裸指针 wrap 一层，NonNull 在类型层面上确保了指针非空，减少了犯错的概率。

bio 类似链表节点，我们可以对其进行 Iterator trait 的封装，使迭代更安全。

```rust
pub struct BioIterator<'a> {
   pub(crate) bio: Option<Bio<'a>>,
}

impl<'a> core::iter::Iterator for BioIterator<'a> {
   type Item = Bio<'a>;

   #[inline(always)]
   fn next(&mut self) -> Option<Bio<'a>> {
       if let Some(current) = self.bio.take() {
           self.bio = current.next();
           Some(current)
       } else {
           None
       }
   }
}
```

### 驱动

待续
