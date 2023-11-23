# Linux blk-mq介绍

Linux blk-mq是Linux块层的一个新框架，它可以让高性能的存储设备通过同时向块设备发送和处理IO请求，从而实现高达数百万的每秒输入/输出操作（IOPS），利用了现代存储设备的并行性。

## blk-mq的主要接口

blk-mq的主要接口有以下几个：

- `blk_mq_alloc_tag_set`：分配一个tag set结构，用于存储blk-mq的配置信息，例如硬件队列的数量，软件队列的数量，请求的大小，驱动程序的回调函数等。
- `blk_mq_init_queue`：根据tag set结构，初始化一个请求队列（request queue），用于管理块设备的IO请求。
- `blk_mq_alloc_disk`：根据请求队列，分配一个磁盘结构（gendisk），用于表示块设备的基本信息，例如设备号，扇区大小，容量等。
- `blk_mq_free_tag_set`：释放一个tag set结构，以及与之关联的资源，例如tag位图，软件队列，硬件队列等。
- `blk_mq_free_disk`：释放一个磁盘结构，以及与之关联的资源，例如请求队列，设备节点等。
- `blk_mq_start_request`：开始处理一个IO请求，将其从软件队列移动到硬件队列，或者直接发送到硬件。
- `blk_mq_complete_request`：完成一个IO请求，将其从硬件队列移除，或者直接返回给用户空间，同时释放tag和资源。
- `blk_mq_map_queue`：根据CPU编号，返回一个对应的硬件队列，用于将IO请求发送到合适的硬件队列。
- `blk_mq_tag_to_rq`：根据tag编号，返回一个对应的IO请求，用于在硬件队列中查找或操作IO请求。

## blk-mq的代码示例

下面是一个使用blk-mq的简单的代码示例，它实现了一个虚拟的块设备，可以在内存中读写数据。

```c
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/blk-mq.h>

#define VBLK_NAME "vblk"
#define VBLK_SECTOR_SIZE 512
#define VBLK_CAPACITY (1024 * 1024 * 1024) // 1 GB
#define VBLK_MAX_REQUESTS 256
#define VBLK_HW_QUEUES 4
#define VBLK_SW_QUEUES 8

static struct vblk_device {
    struct blk_mq_tag_set tag_set;
    struct request_queue *queue;
    struct gendisk *disk;
    u8 *data;
} vblk_dev;

static int vblk_open(struct block_device *bdev, fmode_t mode)
{
    return 0;
}

static void vblk_release(struct gendisk *disk, fmode_t mode)
{
}

static const struct block_device_operations vblk_fops = {
    .owner = THIS_MODULE,
    .open = vblk_open,
    .release = vblk_release,
};

static blk_status_t vblk_queue_rq(struct blk_mq_hw_ctx *hctx,
                  const struct blk_mq_queue_data *bd)
{
    struct request *rq = bd->rq;
    struct bio_vec bvec;
    struct req_iterator iter;
    u64 offset;
    u32 len;
    u8 *buf;

    blk_mq_start_request(rq);

    offset = blk_rq_pos(rq) * VBLK_SECTOR_SIZE;

    // rq_for_each_segment(bvec, rq, iter) {
    //     len = bvec.bv_len;
    //     buf = page_address(bvec.bv_page) + bvec.bv_offset;
    //     if (rq_data_dir(rq) == READ) {
    //         memcpy(buf, vblk_dev.data + offset, len);
    //     } else {
    //         memcpy(vblk_dev.data + offset, buf, len);
    //     }
    //     offset += len;
    // }

    blk_mq_complete_request(rq);

    return BLK_STS_OK;
}

// 定义一个自定义的数据结构，用于存储请求相关的信息
struct vblk_request {
    struct request *rq; // 指向blk-mq的请求结构
    void *data; // 指向自定义的数据缓冲区
    // 其他字段
};

// 实现.complete接口的函数
static void vblk_complete(struct request *rq, blk_status_t status)
{
    // 获取自定义的数据结构
    struct vblk_request *vreq = blk_mq_rq_to_pdu(rq);

    // 释放自定义的数据缓冲区
    kfree(vreq->data);

    // 调用blk-mq的默认完成函数
    blk_mq_end_request(rq, status);
}

static struct blk_mq_ops vblk_mq_ops = {
    .queue_rq = vblk_queue_rq,
    .complete = vblk_complete,
};

static int __init vblk_init(void)
{
    int ret;

    // allocate tag set
    vblk_dev.tag_set.ops = &vblk_mq_ops;
    vblk_dev.tag_set.nr_hw_queues = VBLK_HW_QUEUES;
    vblk_dev.tag_set.queue_depth = VBLK_MAX_REQUESTS;
    vblk_dev.tag_set.numa_node = NUMA_NO_NODE;
    vblk_dev.tag_set.cmd_size = 0;
    vblk_dev.tag_set.flags = BLK_MQ_F_SHOULD_MERGE;
    vblk_dev.tag_set.driver_data = &vblk_dev;

    ret = blk_mq_alloc_tag_set(&vblk_dev.tag_set);
    if (ret) {
        pr_err("Failed to allocate tag set\n");
        return ret;
    }

    // allocate request queue
    vblk_dev.queue = blk_mq_init_queue(&vblk_dev.tag_set);
    if (IS_ERR(vblk_dev.queue)) {
        pr_err("Failed to allocate request queue\n");
        ret = PTR_ERR(vblk_dev.queue);
        goto free_tag_set;
    }

    // allocate disk
    vblk_dev.disk = blk_mq_alloc_disk(&vblk_dev.tag_set, NULL);
    if (IS_ERR(vblk_dev.disk)) {
        pr_err("Failed to allocate disk\n");
        ret = PTR_ERR(vblk_dev.disk);
        goto free_queue;
    }

    // set disk parameters
    vblk_dev.disk->major = 0;
    vblk_dev.disk->first_minor = 0;
    vblk_dev.disk->fops = &vblk_fops;
    vblk_dev.disk->private_data = &vblk_dev;
    vblk_dev.disk->queue = vblk_dev.queue;
    vblk_dev.disk->flags = GENHD_FL_EXT_DEVT;
    sprintf(vblk_dev.disk->disk_name, VBLK_NAME);
    set_capacity(vblk_dev.disk, VBLK_CAPACITY / VBLK_SECTOR_SIZE);

    // allocate data buffer
    vblk_dev.data = vzalloc(VBLK_CAPACITY);
    if (!vblk_dev.data) {
        pr_err("Failed to allocate data buffer\n");
        ret = -ENOMEM;
        goto free_disk;
    }

    // add disk
    add_disk(vblk_dev.disk);

    pr_info("vblk device initialized\n");

    return 0;

free_disk:
    blk_mq_free_disk(vblk_dev.disk);
free_queue:
    blk_cleanup_queue(vblk_dev.queue);
free_tag_set:
    blk_mq_free_tag_set(&vblk_dev.tag_set);
    return ret;
}

static void __exit vblk_exit(void)
{
    // remove disk
    del_gendisk(vblk_dev.disk);

    // free data buffer
    vfree(vblk_dev.data);

    // free disk
    blk_mq_free_disk(vblk_dev.disk);

    // free request queue
    blk_cleanup_queue(vblk_dev.queue);

    // free tag set
    blk_mq_free_tag_set(&vblk_dev.tag_set);

    pr_info("vblk device exited\n");
}

module_init(vblk_init);
module_exit(vblk_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("AAA");
MODULE_DESCRIPTION("A simple example of using blk-mq");
```

## blk-mq的示意图