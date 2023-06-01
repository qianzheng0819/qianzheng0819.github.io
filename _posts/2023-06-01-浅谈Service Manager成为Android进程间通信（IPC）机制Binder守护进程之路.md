---
layout: post
title:  "浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路"
date:   2023-06-1 10:11:00 +0800
categories: android
tags:   Framework
description:
---

前言
----------
老罗的android之旅系列博客，应该是不少人入门framework之选

这系列的博客，除去sufaceFlinger和wms相关，我基本上都已经阅读过2-3次

当然也收获良多，理解了binder机制，理解了activity的启动流程，理解了activity的实现框架等等

在阅读binder源码时，往往有那种恍然大悟的愉悦感

但是阅读过后一个月，记忆里就所剩无多，只剩下类似BpBinder,BnBinder,ProcessState,

IPCThreadState这些重要类的印象。好记忆不如烂笔头，这就是我写下这系列博客的原因。


Service Manager成为Binder守护进程之路
-----------------
既然Service Manager组件是用来管理Server并且向Client提供查询Server远程接口的功能，那么，Service Manager就必然要和Server以及Client进

行通信了。我们知道，Service Manger、Client和Server三者分别是运行在独立的进程当中，这样它们之间的通信也属于进程间通信了，而且也是采

用Binder机制进行进程间通信，因此，Service Manager在充当Binder机制的守护进程的角色的同时，也在充当Server的角色，然而，它是一种特殊

的Server，下面我们将会看到它的特殊之处。

{% highlight cpp %}
int main(int argc, char **argv)
{
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;

    bs = binder_open(128*1024);

    if (binder_become_context_manager(bs)) {
        LOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
{% endhighlight %}

 main函数主要有三个功能:

 - 打开Binder设备文件
 - 告诉Binder驱动程序自己是Binder上下文管理者
 - 进入一个无穷循环，充当Server的角色，等待Client的请求

 宏BINDER_SERVICE_MANAGER定义frameworks/base/cmds/servicemanager/binder.h文件中：
 `#define BINDER_SERVICE_MANAGER ((void*) 0)`

 Service Manager在充当守护进程的同时，它充当Server的角色，当它作为远程接口使用时，它的句柄值便为0.


函数首先是执行打开Binder设备文件的操作binder_open，这个函数位于frameworks/base/cmds/servicemanager/binder.c文件中：
{% highlight cpp %}
struct binder_state *binder_open(unsigned mapsize)
{
    struct binder_state *bs;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return 0;
    }

    bs->fd = open("/dev/binder", O_RDWR);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

        /* TODO: check version */

    return bs;
}
{% endhighlight %}

我们先看open函数，调用的是内核的binder.c文件

#### Binder.open()
{% highlight cpp %}
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc; // binder进程 【见附录3.1】

    proc = kzalloc(sizeof(*proc), GFP_KERNEL); // 为binder_proc结构体在分配kernel内存空间
    if (proc == NULL)
        return -ENOMEM;
    get_task_struct(current);
    proc->tsk = current;   //将当前线程的task保存到binder进程的tsk
    INIT_LIST_HEAD(&proc->todo); //初始化todo列表
    init_waitqueue_head(&proc->wait); //初始化wait队列
    proc->default_priority = task_nice(current);  //将当前进程的nice值转换为进程优先级

    binder_lock(__func__);   //同步锁，因为binder支持多线程访问
    binder_stats_created(BINDER_STAT_PROC); //BINDER_PROC对象创建数加1
    hlist_add_head(&proc->proc_node, &binder_procs); //将proc_node节点添加到binder_procs为表头的队列
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death); //初始化已分发的死亡通知列表
    filp->private_data = proc;       //file文件指针的private_data变量指向binder_proc数据
    binder_unlock(__func__); //释放同步锁

    return 0;
}
{% endhighlight %}
具体流程，注释里已经写的很清楚了

#### Binder.mmap()
{% highlight cpp %}
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct vm_struct *area; //内核虚拟空间
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;  //【见附录3.9】

    if (proc->tsk != current)
        return -EINVAL;

    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;  //保证映射内存大小不超过4M

    mutex_lock(&binder_mmap_lock);  //同步锁
    //采用IOREMAP方式，分配一个连续的内核虚拟空间，与进程虚拟空间大小一致
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
    proc->buffer = area->addr; //指向内核虚拟空间的地址
    //地址偏移量 = 用户虚拟地址空间 - 内核虚拟地址空间
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    mutex_unlock(&binder_mmap_lock); //释放锁

    ...
    //分配物理页的指针数组，数组大小为vma的等效page个数；
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    //分配物理页面，同时映射到内核空间和进程空间，先分配1个物理页 【见小节2.3.1】
    if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
        ret = -ENOMEM;
        failure_string = "alloc small buf";
        goto err_alloc_small_buf_failed;
    }
    buffer = proc->buffer; //binder_buffer对象 指向proc的buffer地址
    INIT_LIST_HEAD(&proc->buffers); //创建进程的buffers链表头
    list_add(&buffer->entry, &proc->buffers); //将binder_buffer地址 加入到所属进程的buffers队列
    buffer->free = 1;
    //将空闲buffer放入proc->free_buffers中
    binder_insert_free_buffer(proc, buffer);
    //异步可用空间大小为buffer总大小的一半。
    proc->free_async_space = proc->buffer_size / 2;
    barrier();
    proc->files = get_files_struct(current);
    proc->vma = vma;
    proc->vma_vm_mm = vma->vm_mm;
    return 0;

    ...// 错误flags跳转处，free释放内存之类的操作
    return ret;
}
{% endhighlight %}

这里为什么会同时使用进程虚拟地址空间和内核虚拟地址空间来映射同一个物理页面呢？这就是Binder进程间通信机制的精髓所在了，同一个物理页面，

一方映射到进程虚拟地址空间，一方面映射到内核虚拟地址空间，这样，进程和内核之间就可以减少一次内存拷贝了，提到了进程间通信效率。举个例子如

，Client要将一块内存数据传递给Server，一般的做法是，Client将这块数据从它的进程空间拷贝到内核空间中，然后内核再将这个数据从内核空间拷贝

到Server的进程空间，这样，Server就可以访问这个数据了。但是在这种方法中，执行了两次内存拷贝操作，而采用我们上面提到的方法，只需要

把Client进程空间的数据拷贝一次到内核空间，然后Server与内核共享这个数据就可以了，整个过程只需要执行一次内存拷贝，提高了效率。
