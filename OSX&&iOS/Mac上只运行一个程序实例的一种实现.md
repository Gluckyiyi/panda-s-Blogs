# Mac上只运行一个程序实例的一种实现

## 前言

在开发应用程序的时候，我们难免要实现确保用户的PC上只能运行程序的一个实例，实现的方式肯定不止一种，这里我就分享我在项目中(Mac的CLI程序)使用的一种方式——锁文件的方式。

## Show me the code

废话不多说，直接上代码。

`int lockfile(int ff){`

struct flock f1;

fl.l_type = F_WRLCK;

​    fl.l_start = 0;    

​    fl.l_whence = SEEK_SET;

​    fl.l_len = 0;

​    **return**(fcntl(ff,F_SETLK,&fl));

`}`

> 注意：在mac上使用的时候需要导入<fcntl.h>和<sys/file.h>头文件。

### 代码简要分析

1. 首先是flock这个结构体，这个结构体的定义如下

   `struct flock {`

   ​	`off_t   l_start;`       

   ​	`off_t   l_len;`        

   ​	`pid_t   l_pid;`       

   ​	`short   l_type;`      

   ​	`short   l_whence;`  

   `};`

   **l_start**： 是设置锁定区域的开头位置

   **l_len:** 是设置锁定区域的大小

   **l_pid：设置锁定动作的进程

   **l_type：**设置锁定的状态。它有三种状态：**F_RDLCK**表示共享锁；**F_WRLCK**表示排它锁；**F_UNLCK**是删除之前建立的锁定。

   **l_whence：**是决定**l_start**的位置。它有三种状态：**SEEK_SET**是以文件开头为锁定的起始位置；**SEEK_CUR**是以目前文件读写位置为锁定的起始位置；**SEEK_END**是以文件结尾为锁定的起始位置。

   

   2.在本例中，将 **fl.l_start**设置为0，**l_type**设置为**F_WRLCK**，**l_whence**设置为**SEEK_SET**，实际上是对文件加了一个从文件开头的排它锁，**l_len**我设置了0，是锁定了整个文件。

   在最最重要的锁文件环节我使用了fcntl函数，其实flock也可以锁定文件，**但fcntl()是唯一的符合POSIX标准的文件锁实现，所以也是唯一可移植的，功能比较强大。**建议使用。

   fcntl函数的返回值是：若成功则返回0，否则返回-1.

   ### 实现程序的单例运行

   基于上面的思路，其实我们只要在程序运行之初的时候，调用open函数打开文件(文件存在的时候)或者创建文件(文件不存在，程序第一次运行的时候)，然后在调用lock方法，根据返回值判断只要返回值是非0，那么则表示程序已经运行了，此时不在运行程序，直接终止即可了。

   代码如下

    `int fd = open("/tmp/test.pid", O_RDWR|O_CREAT,S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH);`

   ​        `if (lockfile(fd) !=0) {`

   ​            `NSLog(@"程序已经运行了..");`

   ​            `return 0;`

   ​        `}`

   

   ## 结尾

   内容有什么错误和疏忽的地方希望大家积极指出，更好的实现方式也希望可以得到赐教！

   

   