

##### 命令

- lsmod：查看已经加载的内核模块， 通过读取文件/proc/modules来获取其信息 
- 加载内核模块：
  - 如果模块初始化函数返回负值会加载失败提示权限不够
  - modprobe 加载模块名字，自己去找依赖的模块， 它首先在文件/etc/modprobe.conf中查找该字符串 ， 接下来，modprobe查看文件 /lib/modules/version/modules.dep ，以查看是否必须加载其他模块才能加载所请求的模块。 该文件由depmod -a创建，包含模块依赖项 
  - insmod  直接加载ko文件，需要自己提前添加依赖模块
  - insmod要求你传递完整的路径名并以正确的顺序插入模块，而modprobe只取名字，没有任何扩展名，并通过解析/ lib找出它需要知道的所有内容/modules/version/modules.dep 。 
-  modinfo：查看内核模块信息
-  rmmod：删除内核模块
-  dmesg ：查看打印信息

##### 包含头文件

```c
#include <linux/module.h>   /* Needed by all modules */
#include <linux/kernel.h>   /* Needed for KERN_INFO */
#include <linux/init.h>     /* Needed for the macros */

#include <linux/moduleparam.h>
#include <linux/stat.h>

```

##### printk

​		 printk（）并不是要向用户传达信息， 它是内核的日志记录机制，用于记录信息或发出警告。 因此，每个printk（）语句都带有一个优先级，即您看到的<1>和KERN_ALERT 。 有8个优先级，内核有宏，所以你不必使用神秘的数字，你可以在linux / kernel.h中查看它们（及其含义）。 如果未指定优先级，则将使用默认优先级DEFAULT_MESSAGE_LOGLEVEL 。 

​		 如果优先级小于int console_loglevel ，则会在当前终端上打印该消息。 如果syslogd和klogd都在运行，那么该消息也会附加到/ var / log / messages ，无论它是否打印到控制台。 我们使用高优先级（如KERN_ALERT ）来确保将printk（）消息打印到控制台而不是仅记录到日志文件中。 编写实际模块时，您需要使用对当前情况有意义的优先级 

```c
#define KERN_EMERG   0     /*紧急事件消息，系统崩溃之前提示，表示系统不可用*/
#define KERN_ALERT   1     /*报告消息，表示必须立即采取措施*/
#define KERN_CRIT    2     /*临界条件，通常涉及严重的硬件或软件操作失败*/
#define KERN_ERR     3     /*错误条件，驱动程序常用KERN_ERR来报告硬件的错误*/
#define KERN_WARNING 4     /*警告条件，对可能出现问题的情况进行警告*/
#define KERN_NOTICE  5     /*正常但又重要的条件，用于提醒*/
#define KERN_INFO    6     /*提示信息，如驱动程序启动时，打印硬件信息*/
#define KERN_DEBUG   7     /*调试级别的消息*/
```



##### 最简单内核模块

- 内核2.1.3以前，只能使用这两个名字

```c
/*  
 *  hello-1.c - The simplest kernel module.
 */
#include <linux/module.h>   /* Needed by all modules */
#include <linux/kernel.h>   /* Needed for KERN_INFO */

int init_module(void) //初始化函数
{
    printk(KERN_INFO "Hello world 1.\n");

    /* 
     * A non 0 return means init_module failed; module can't be loaded. 
     */
    return 0;
}

void cleanup_module(void) //退出函数
{
    printk(KERN_INFO "Goodbye world 1.\n");
}
```

- 新版本，使用module_init 和 module_exit 去声明

```c
#include <linux/module.h>   /* Needed by all modules */
#include <linux/kernel.h>   /* Needed for KERN_INFO */
#include <linux/init.h>     /* Needed for the macros */

static int __init hello_2_init(void)
{
    printk(KERN_INFO "Hello, world 2\n");
    return 0;
}

static void __exit hello_2_exit(void)
{
    printk(KERN_INFO "Goodbye, world 2\n");
}

module_init(hello_2_init);
module_exit(hello_2_exit);

MODULE_LICENSE("GPL") //许可等级
MODULE_DESCRIPTION("Description") //模块的介绍
MODULE_AUTHOR("wang") //作者
```

```c
/*
 * The following license idents are currently accepted as indicating free
 * software modules
 *
 *  "GPL"               [GNU Public License v2 or later]
 *  "GPL v2"            [GNU Public License v2]
 *  "GPL and additional rights" [GNU Public License v2 rights and more]
 *  "Dual BSD/GPL"          [GNU Public License v2
 *                   or BSD license choice]
 *  "Dual MIT/GPL"          [GNU Public License v2
 *                   or MIT license choice]
 *  "Dual MPL/GPL"          [GNU Public License v2
 *                   or Mozilla license choice]
 *
 * The following other idents are available
 *
 *  "Proprietary"           [Non free products]
 *
 * There are dual licensed components, but when running with Linux it is the
 * GPL that is relevant so this is a non issue. Similarly LGPL linked with GPL
 * is a GPL combined work.
 *
 * This exists for several reasons
 * 1.   So modinfo can show license info for users wanting to vet their setup 
 *  is free
 * 2.   So the community can ignore bug reports including proprietary modules
 * 3.   So vendors can do likewise based on their own policies
 */
```

##### 参数的传递

​		 要允许将参数传递给模块，请声明将命令行参数的值作为全局变量的变量，然后使用module_param（）宏（在linux / moduleparam.h中定义）来设置机制 。 在运行时，insmod将使用给定的任何命令行参数填充变量，例如./insmod mymodule.ko myvariable = 5 。 

​		为清楚起见，变量声明和宏应放在模块的开头。 示例代码应该清除我公认的糟糕解释。 

###### 基础语法

```c
/*
*  module_para()   宏有3个参数：变量的名称，它在sysfs中对应文件的类型和权限
*  module_param_array()
*  module_param_string()
*  MODULE_PARM_DESC()  用于记录模块可以采用的参数。 它需要两个参数：变量名称和描述该变量的自由格式字符串。
*/
int myint = 3;
module_param(myint, int, 0);
MODULE_PARM_DESC(myint, "A integer");

int myintarray[2];
module_param_array(myintarray, int, NULL, 0); /* not interested in count */
MODULE_PARM_DESC(myintArray, "An array of integers");

int myshortarray[4];
int count;
module_parm_array(myshortarray, short,&count, 0); /* put count into "count" variable */
```

###### 权限

```
S_IRUSR
  Permits the file's owner to read it.

S_IWUSR
  Permits the file's owner to write to it.

S_IRGRP
  Permits the file's group to read it.

S_IWGRP
  Permits the file's group to write to it.

```

###### 示例

```c
/*
 *  hello-5.c - Demonstrates command line argument passing to a module.
 */
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Peter Jay Salzman");

static short int myshort = 1;
static int myint = 420;
static long int mylong = 9999;
static char *mystring = "blah";
static int myintArray[2] = { -1, -1 };
static int arr_argc = 0;

/* 
 * module_param(foo, int, 0000)
 * The first param is the parameters name
 * The second param is it's data type
 * The final argument is the permissions bits, 
 * for exposing parameters in sysfs (if non-zero) at a later stage.
 */

module_param(myshort, short, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
MODULE_PARM_DESC(myshort, "A short integer");
module_param(myint, int, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
MODULE_PARM_DESC(myint, "An integer");
module_param(mylong, long, S_IRUSR);
MODULE_PARM_DESC(mylong, "A long integer");
module_param(mystring, charp, 0000);
MODULE_PARM_DESC(mystring, "A character string");

/*
 * module_param_array(name, type, num, perm);
 * The first param is the parameter's (in this case the array's) name
 * The second param is the data type of the elements of the array
 * The third argument is a pointer to the variable that will store the number 
 * of elements of the array initialized by the user at module loading time
 * The fourth argument is the permission bits
 */
module_param_array(myintArray, int, &arr_argc, 0000);
MODULE_PARM_DESC(myintArray, "An array of integers");

static int __init hello_5_init(void)
{
    int i;
    printk(KERN_INFO "Hello, world 5\n=============\n");
    printk(KERN_INFO "myshort is a short integer: %hd\n", myshort);
    printk(KERN_INFO "myint is an integer: %d\n", myint);
    printk(KERN_INFO "mylong is a long integer: %ld\n", mylong);
    printk(KERN_INFO "mystring is a string: %s\n", mystring);
    for (i = 0; i < (sizeof myintArray / sizeof (int)); i++)
    {
        printk(KERN_INFO "myintArray[%d] = %d\n", i, myintArray[i]);
    }
    printk(KERN_INFO "got %d arguments for myintArray.\n", arr_argc);
    return 0;
}

static void __exit hello_5_exit(void)
{
    printk(KERN_INFO "Goodbye, world 5\n");
}

module_init(hello_5_init);
module_exit(hello_5_exit);
```

##### Makefile

```makefile
obj-m += hello-1.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

##### 设备驱动程序

​		 一类模块是设备驱动程序，它为电视卡或串行端口等硬件提供功能。 在unix上，每个硬件都由位于/ dev中的文件表示，该文件命名为设备文件 ，该文件提供与硬件通信的方法。 设备驱动程序代表用户程序提供通信。 

​		ls -l /dev   可以看到所以的设备驱动，有两个设备编号： 设备的主要编号、从编号。 主要编号告诉您使用哪个驱动程序访问硬件。 为每个驱动程序分配一个唯一的主编号; 具有相同主要编号的所有设备文件由同一驱动程序控制， 驱动程序使从编号来区分它控制的各种硬件。

###### 设备驱动的种类

​		 块设备：块设备具有请求缓冲区，因此它们可以选择响应请求的最佳顺序。 这在存储设备的情况下是重要的，其中读取或写入彼此接近的扇区更快，而不是那些相距更远的扇区 

​		字符设备： 能接受块中的输入和返回输出（其大小可以根据设备而变化），而字符设备允许使用它们喜欢的尽可能多的字节 

​		使用ls -l 查看第一个字母是C 就是字符设备，是B就是是块设备

​		 如果要查看已分配的主要编号，可以查看 / proc / devices 

###### 设备驱动的安装

​	 设备文件都是由mknod命令创建的。 要创建一个名为“coffee”且主要/次要编号为12和2的新char设备，只需执行mknod / dev / coffee c 12 2即可 。 您不必将设备文件放入/ dev ，但它是按惯例完成的。 Linus将他的设备文件放在/ dev中 ，所以你应该这样做。 但是，在创建用于测试目的的设备文件时，可以将它放在编译内核模块的工作目录中。 完成编写设备驱动程序后，请务必将其放在正确的位置 

###### 字符设备驱动

​		file_operations 结构体：/fs.h

```c
struct file_operations {
    struct module *owner;
     loff_t(*llseek) (struct file *, loff_t, int);
     ssize_t(*read) (struct file *, char __user *, size_t, loff_t *);
     ssize_t(*aio_read) (struct kiocb *, char __user *, size_t, loff_t);
     ssize_t(*write) (struct file *, const char __user *, size_t, loff_t *);
     ssize_t(*aio_write) (struct kiocb *, const char __user *, size_t,
                  loff_t);
    int (*readdir) (struct file *, void *, filldir_t);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    int (*ioctl) (struct inode *, struct file *, unsigned int,
              unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, struct dentry *, int datasync);
    int (*aio_fsync) (struct kiocb *, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
     ssize_t(*readv) (struct file *, const struct iovec *, unsigned long,
              loff_t *);
     ssize_t(*writev) (struct file *, const struct iovec *, unsigned long,
               loff_t *);
     ssize_t(*sendfile) (struct file *, loff_t *, size_t, read_actor_t,
                 void __user *);
     ssize_t(*sendpage) (struct file *, struct page *, int, size_t,
                 loff_t *, int);
    unsigned long (*get_unmapped_area) (struct file *, unsigned long,
                        unsigned long, unsigned long,
                        unsigned long);
};
```

注册设备

```c
/*
*    major 主设备号
*    name  设备名字
*    fops  设备驱动函数
*/
int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);
```

 		将主要编号0传递给register_chrdev ，则返回值将是动态分配的主编号。 缺点是您无法提前制作设备文件，因为您不知道主要编号是什么。 有几种方法可以做到这一点。 首先，驱动程序本身可以打印新分配的号码，我们可以手动制作设备文件。 其次，新注册的设备将在/ proc / devices中有一个条目，我们可以手工制作设备文件，也可以编写shell脚本来读取文件并制作设备文件。 第三种方法是我们可以让我们的驱动程序在成功注册后使用mknod系统调用来生成设备文件，并在调用cleanup_module期间使用rm。 

###### 取消设备

​		 当root感觉它时，我们不能允许内核模块被rmmod编辑。 如果一个进程打开了设备文件，然后我们删除了内核模块，那么使用该文件会导致调用以前使用相应函数（读/写）的内存位置。 如果我们很幸运，那里没有加载其他代码，我们会收到一条丑陋的错误消息。 如果我们运气不好，另一个内核模块被加载到同一个位置，这意味着跳转到内核中另一个函数的中间。 这样的结果是不可能预测的，但它们不是非常积极的。 

​		 通常，当您不想允许某些内容时，您将从应该执行此操作的函数返回错误代码（负数）。 使用cleanup_module是不可能的，因为它是一个void函数。 但是，有一个计数器可以跟踪使用模块的进程数量。 通过查看/ proc / modules的第3个字段，您可以看到它的价值。 如果此数字不为零，则rmmod将失败。 请注意，您不必在cleanup_module中检查计数器，因为将通过linux / module.c中定义的系统调用sys_delete_module为您执行检查。 你不应该直接使用这个计数器，但linux / module.h中定义了一些函数，可以增加，减少和显示这个计数器： 

```c
unregister_chrdev(Major, DEVICE_NAME);
try_module_get（THIS_MODULE） //增加使用次数。
module_put（THIS_MODULE）     //减少使用次数。
```

示例

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <asm/uaccess.h>    /* for put_user */

/*  
 *  Prototypes - this would normally go in a .h file
 */
int init_module(void);
void cleanup_module(void);
static int device_open(struct inode *, struct file *);
static int device_release(struct inode *, struct file *);
static ssize_t device_read(struct file *, char *, size_t, loff_t *);
static ssize_t device_write(struct file *, const char *, size_t, loff_t *);

#define SUCCESS 0
#define DEVICE_NAME "chardev"   /* Dev name as it appears in /proc/devices   */
#define BUF_LEN 80      /* Max length of the message from the device */

/* 
 * Global variables are declared as static, so are global within the file. 
 */

static int Major;       /* Major number assigned to our device driver */
static int Device_Open = 0; /* Is device open?  
                 * Used to prevent multiple access to device */
static char msg[BUF_LEN];   /* The msg the device will give when asked */
static char *msg_Ptr;

static struct file_operations fops = {
    .read = device_read,
    .write = device_write,
    .open = device_open,
    .release = device_release
};

/*
 * This function is called when the module is loaded
 */
int init_module(void)
{
        Major = register_chrdev(0, DEVICE_NAME, &fops);

    if (Major < 0) {
      printk(KERN_ALERT "Registering char device failed with %d\n", Major);
      return Major;
    }

    printk(KERN_INFO "I was assigned major number %d. To talk to\n", Major);
    printk(KERN_INFO "the driver, create a dev file with\n");
    printk(KERN_INFO "'mknod /dev/%s c %d 0'.\n", DEVICE_NAME, Major);
    printk(KERN_INFO "Try various minor numbers. Try to cat and echo to\n");
    printk(KERN_INFO "the device file.\n");
    printk(KERN_INFO "Remove the device file and module when done.\n");

    return SUCCESS;
}

/*
 * This function is called when the module is unloaded
 */
void cleanup_module(void)
{
    /* 
     * Unregister the device 
     */
    int ret = unregister_chrdev(Major, DEVICE_NAME);
    if (ret < 0)
        printk(KERN_ALERT "Error in unregister_chrdev: %d\n", ret);
}

/*
 * Methods
 */

/* 
 * Called when a process tries to open the device file, like
 * "cat /dev/mycharfile"
 */
static int device_open(struct inode *inode, struct file *file)
{
    static int counter = 0;

    if (Device_Open)
        return -EBUSY;

    Device_Open++;
    sprintf(msg, "I already told you %d times Hello world!\n", counter++);
    msg_Ptr = msg;
    try_module_get(THIS_MODULE);

    return SUCCESS;
}

/* 
 * Called when a process closes the device file.
 */
static int device_release(struct inode *inode, struct file *file)
{
    Device_Open--;      /* We're now ready for our next caller */

    /* 
     * Decrement the usage count, or else once you opened the file, you'll
     * never get get rid of the module. 
     */
    module_put(THIS_MODULE);

    return 0;
}

/* 
 * Called when a process, which already opened the dev file, attempts to
 * read from it.
 */
static ssize_t device_read(struct file *filp,   /* see include/linux/fs.h   */
               char *buffer,    /* buffer to fill with data */
               size_t length,   /* length of the buffer     */
               loff_t * offset)
{
    /*
     * Number of bytes actually written to the buffer 
     */
    int bytes_read = 0;

    /*
     * If we're at the end of the message, 
     * return 0 signifying end of file 
     */
    if (*msg_Ptr == 0)
        return 0;

    /* 
     * Actually put the data into the buffer 
     */
    while (length && *msg_Ptr) {

        /* 
         * The buffer is in the user data segment, not the kernel 
         * segment so "*" assignment won't work.  We have to use 
         * put_user which copies data from the kernel data segment to
         * the user data segment. 
         */
        put_user(*(msg_Ptr++), buffer++);

        length--;
        bytes_read++;
    }

    /* 
     * Most read functions return the number of bytes put into the buffer
     */
    return bytes_read;
}

/*  
 * Called when a process writes to dev file: echo "hi" > /dev/hello 
 */
static ssize_t
device_write(struct file *filp, const char *buff, size_t len, loff_t * off)
{
    printk(KERN_ALERT "Sorry, this operation isn't supported.\n");
    return -EINVAL;
}	
```

