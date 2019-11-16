

##### 命令

- ismod：查看已经加载的内核模块， 通过读取文件/proc/modules来获取其信息 
- 加载内核模块：
  -  modprobe 加载模块名字，自己去找依赖的模块， 它首先在文件/etc/modprobe.conf中查找该字符串 ， 接下来，modprobe查看文件 /lib/modules/version/modules.dep ，以查看是否必须加载其他模块才能加载所请求的模块。 该文件由depmod -a创建，包含模块依赖项 
  -  insmod  直接加载ko文件，需要自己提前添加依赖模块
  -  nsmod要求你传递完整的路径名并以正确的顺序插入模块，而modprobe只取名字，没有任何扩展名，并通过解析/ lib找出它需要知道的所有内容/modules/version/modules.dep 。 
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

