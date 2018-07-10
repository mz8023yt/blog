---
title: '[CAR] 学习总结(三) 字符设备驱动框架'
date: 2018-07-10 21:50:48
tags:
  - train
categories: Experience
---

开发环境搭建好了，现在开始做任务吧。

### 任务要求

> 编写一个基本字符设备，验证字符设备常用接口：open/close/read/write/ioctl

### 任务实现

驱动模块：`first_drv.c`

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <asm/uaccess.h>        /* copy_to_user ... */
#include <linux/ioctl.h>

/* simple log */
#define log(fmt, arg...)\
        printk(KERN_INFO "[Paul][%s][%d] "fmt"\n", __func__, __LINE__, ##arg);

#define FIRST_IOC_MAGIC 'F'
#define FIRST_ONLY_EXECUTE _IO(FIRST_IOC_MAGIC, 0)
#define FIRST_ONLY_WRITE   _IOW(FIRST_IOC_MAGIC, 1, int)
#define FIRST_ONLY_READ    _IOR(FIRST_IOC_MAGIC, 2, int)
#define FIRST_RW           _IOWR(FIRST_IOC_MAGIC, 3, int)
#define FIRST_IOCTL_MAXNR  3

int first_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
        if (_IOC_TYPE(cmd) != FIRST_IOC_MAGIC) {
                log("error1");
                return -ENOTTY;
        }
        
        if (_IOC_NR(cmd) > FIRST_IOCTL_MAXNR) {
                log("error2");
                return -ENOTTY;
        }

        switch(cmd)
        {
                case FIRST_ONLY_EXECUTE:
                        log("FIRST_ONLY_EXECUTE");
                        break;
                case FIRST_ONLY_WRITE:
                        log("FIRST_ONLY_WRITE");
                        break;
                case FIRST_ONLY_READ:
                        log("FIRST_ONLY_READ");
                        break;
                case FIRST_RW:
                        log("FIRST_RW");
                        break;
                default:
                        log("default");
        }
}

static int first_open(struct inode *my_indoe, struct file *my_file)
{
        log("standard open ok");
        return 0;
}

static int first_release(struct inode *my_indoe, struct file *my_file)
{
        log("standard release ok");
        return 0;
}

static ssize_t first_read(struct file *my_file, char __user *buff, size_t cnt, loff_t *loff)
{
        int tmp = 128;
        copy_to_user(buff, &tmp, 4);
        log("standard read ok");
        return 0;
}

static ssize_t first_write(struct file *my_file, const char __user *buff, size_t cnt, loff_t *loff)
{
        int tmp = 0;
        copy_from_user(&tmp, buff, 4);
        log("tmp = %d", tmp);
        log("standard write ok");
        return 0;
}

static struct file_operations first_fops =
{
        .open    = first_open,
        .release = first_release,
        .read    = first_read,
        .write   = first_write,
        .unlocked_ioctl = first_ioctl,
};

static int first_major;
static int first_minor;
static dev_t first_devno;

struct cdev * standard_device;
static struct class * first_class;

static int __init mod_init(void)
{
        int ret;
        ret = alloc_chrdev_region(&first_devno, first_minor, 1, "standard_device");
        first_major = MAJOR(first_devno);
        if (ret < 0)
        {
                log("cannot get first_major %d", first_major);
                return -1;
        }

        standard_device = cdev_alloc();
        if (standard_device != NULL)
        {
                cdev_init(standard_device, &first_fops);
                standard_device->owner = THIS_MODULE;
                if (cdev_add(standard_device, first_devno, 1) != 0)
                {
                        log("add cdev error!");
                        goto error;
                }
        }
        else
        {
                log("cdev_alloc error!");
                return -1;
        }

        first_class = class_create(THIS_MODULE, "first_class");
        if (IS_ERR(first_class))
        {
                log("create class error");
                return -1;
        }
        device_create(first_class, NULL, first_devno, NULL, "standard_device");

        log("mod_init ok");
        return 0;

error:
        unregister_chrdev_region(first_devno, 1);
        return ret;
}

static void __exit mod_exit(void)
{
        cdev_del(standard_device);
        unregister_chrdev_region(first_devno, 1);
        device_destroy(first_class, first_devno);
        class_destroy(first_class);
        log("mod_exit ok");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

模块对应 Makefile：

```
obj-m += first_drv.o
KERNEL = /home/baohua/develop/linux

all:
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -C $(KERNEL) M=`pwd` modules
	@echo ">>>>>> make all successful <<<<<<"

clean:
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -C $(KERNEL) M=`pwd` modules clean
	@echo ">>>>>> make clean successful <<<<<<"
```

测试程序：`first_app.c`

```
int main(int argc, char**argv)
{
        int fd;
        printf("[app] O_RDWR = %d\n", O_RDWR);
        fd = open("/dev/standard_device", O_RDWR);

        if(fd < 0)
        {
                printf("[app] driver open fail\n");
                return -1;
        }

        int write_buf = 100;
        write(fd, &write_buf, 4);

        int read_buf = 0;
        read(fd, &read_buf, 4);

        ioctl(fd, FIRST_ONLY_EXECUTE);
        ioctl(fd, FIRST_ONLY_WRITE, 100);
        ioctl(fd, FIRST_ONLY_READ, 100);
        ioctl(fd, FIRST_RW, 100);

        close(fd);
        printf("[app] read_buf = %d\n", read_buf);

        return 0;
}
```

### 任务验证

编译模块和测试程序并拷贝到开发板上：

```
# 编译模块
baohua@baohua-VirtualBox:~/develop/wangbing/01.char$ make

# 编译测试程序
baohua@baohua-VirtualBox:~/develop/wangbing/01.char$ arm-linux-gnueabi-gcc first_app.c -o app

# 挂载开发板根目录
baohua@baohua-VirtualBox:~/develop/wangbing//01.char$ sudo mount -o loop,offset=1048576 /home/baohua/develop/linux/extra/vexpress.img /home/baohua/develop/linux/extra/img

# 拷贝到开发板上
baohua@baohua-VirtualBox:~/develop/wangbing/01.char$ sudo cp first_drv.ko /home/baohua/develop/linux/extra/img/ -a
baohua@baohua-VirtualBox:~/develop/wangbing/01.char$ sudo cp app /home/baohua/develop/linux/extra/img/ -a
```

开发板执行效果图：

![image.png](https://upload-images.jianshu.io/upload_images/11006334-501297bd4f695c67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





