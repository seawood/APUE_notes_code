# Chapter 1. UNIX 基础知识
>从程序员的角度简要地浏览UNIX提供的服务、概念、术语
## UNIX体系结构

>严格意义：操作系统=内核

>广义：操作系统=内核+一些其他软件（system utility、应用程序、shell、公用函数库等）

应用程序可以使用的接口：

- 系统调用：内核的接口
- 公用函数库：构建在系统调用之上
- shell:一个特殊的应用程序，构建在系统调用之上
## 登录
- /etc/passwd
- Linux必备的两种Shell:Bourne shell(/bin/sh)和Bourne-again shell(/bin/bash)
## 文件和目录
> dirent.h: opendir(),readdir(), closedir()

```c
//1_4_1.c
//list file names of a given directory
#include "apue.h"
#include <dirent.h>

int main (int argc, char *argv[])
{
   DIR *dp;
   struct dirent *dirp;
   if (argc != 2) 
      err_quit ("usage: ls directory_name");
   if ((dp = opendir(argv[1])) == NULL)
      err_sys ("can't open %s", argv[1]);
   while ((dirp = readdir(dp)) != NULL)
      printf ("%s\n", dirp->d_name);
  closedir (dp);
  exit(0);
}
```
```c
 struct dirent {
                   ino_t          d_ino;       /* Inode number */
                   off_t          d_off;       /* Not an offset; see below */
                   unsigned short d_reclen;    /* Length of this record */
                   unsigned char  d_type;      /* Type of file; not supported
                                                  by all filesystem types */
                   char           d_name[256]; /* Null-terminated filename */
           };

```
## 输入和输出
- 文件描述符：一个小的非负整数
- 标准输入/输出/错误：是三个特殊的文件描述符，当运行一个新程序时，shell为其打开这3个文件描述符，默认连接到终端
- 两种I/O:
    - 不带缓冲的I/O: 如open、read、write、lseek、close等函数，使用文件描述符，定义在unistd.h文件中，STDIN_FILENO标准输入文件描述符常量，STDOUT_FILENO标准输出文件描述符常量。
    - 标准I/O: 带缓冲，定义在stdio.h文件中，如getc、putc、printf等函数，stdin标准输入常量，stdout标准输出常量。

```c
//copy data from standard input to standard output using I/O without buffer
#include "apue.h"

#define BUFFSIZE 4096

int
main (void)
{
    int n;
    char buf[BUFFSIZE];
    while ((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0)
        if (write(STDOUT_FILENO, buf, n) != n)
                err_sys("write error");
    if (n < 0)
        err_sys("read error");
    exit(0);
}
```
```c
//copy data from standard input to standard ourput using I/O with buffer
#include "apue.h"

int
main (void)
{
   int c;
   while((c = getc(stdin)) != EOF) 
    if((putc(stdout)) == EOF)
        err_sys ("output error");
    if (ferror(stdin))
        err_sys("input error");
    exit(0);
}
```
## 程序和进程
- 程序： 一个储存在磁盘上某个目录中的可执行文件
- 进程：程序通过exec函数读入内存并执行之后就是一个进程，即程序的执行实例
- 进程ID：getpid（）返回进程ID，pid_t类型，保证能够存在一个长整型中
- 线程：一个进程可以有多个线程，线程ID只在它所属的进程内起作用
- 进程控制：fork(),exec(),waitpid()
```c
//read command from standard input and execute
#include "apue.h"
#include <sys/wait.h>

int main(void)
{
   char buf[MAXLINE];
   pid_t pid;
   int status;
   printf("%% ");
   while(fgets(buf, MAXLINE, stdin) != NULL) {
    if (buf[strlen(buf)-1] == '\n')
        buf[strlen(buf)-1] = 0;
    if((pid = fork()) < 0)
        err_sys("fork error");
    else if (pid == 0) {
        execlp(buf, buf, (char*)0);
        err_ret("couldn't execute: %s", buf);
        exit(127);
    }
    if ((pid == waitpid(pid, &status, 0)) < 0) 
        err_sys("waitpid error");
    printf("%% ");
   }
    exit(0);
}
```
## 出错处理
- 当UNIX系统函数出错时，通常返回一个负值(对于返回指向对象指针的函数，出错时返回null指针），而且整型变量errno通常被设置为具有特定信息的值
- 对于errno的值：
    - 如果没有出错，其值不会被清除
    - 任何函数都不会将errno的值设置为0
- 打印出错信息：
```c
#include <string.h>
char *strerror(int errnum);

#include <stdio.h>
void perror(const char *msg);
```
```c
#include "apue.h"
#include <errno.h>
int
main(int argc, char *argv[]) {
   fprintf(stderr, "EACCES: %s\n", strerror(EACCES));
   errno = ENOENT;
   perror(argv[0]);
   exit(0);
}
```
- 出错处理：致命性和非致命性
## 信号
>信号用于通知进程发生了某种情况

对信号的处理方式：

- 忽略信号：
- 按系统默认方式:终止该进程
- 提供一个函数:提供自编的函数,通过signal函数绑定

产生信号的方法： 

- 终端键盘：中断键(Delete/Ctrl-C)/退出键(quit/Ctrl-\\)
- kill函数
```c
#include "apue.h"
#include <sys/wait.h>

static void sig_int(int);

int main(void)
{
   char buf[MAXLINE];
   pid_t pid;
   int status;

   if (signal(SIGINT, sig_int) == SIG_ERR)
    err_sys("signal error");

   printf("%% ");
   while(fgets(buf, MAXLINE, stdin) != NULL) {
    if (buf[strlen(buf)-1] == '\n')
        buf[strlen(buf)-1] = 0;
    if((pid = fork()) < 0)
        err_sys("fork error");
    else if (pid == 0) {
        execlp(buf, buf, (char*)0);
        err_ret("couldn't execute: %s", buf);
        exit(127);
    }
    if ((pid == waitpid(pid, &status, 0)) < 0)
        err_sys("waitpid error");
    printf("%% ");
   }
    exit(0);
}
void
sig_int(int signo)
{
    printf("interrupt\n%%");
}
```
## 时间值

UNIX系统上的两种不同时间值：

- 日历时间：从1970年1月1日00:00:00:00到现在的秒数累计值，由time_t类型保存
- 进程时间（CPU时间）：用于度量进程使用的中央处理器资源，以时钟滴答计算（通过CLOCKS_PER_SEC查看每秒包含多少个时钟滴答），由clock_t类型保存
    + 时钟时间:进程运行的时间总量
    + 用户CPU时间:执行用户指令所用的时间总量
    + 系统CPU时间:为该进程执行内核程序所经历的时间

## 系统调用和库函数
- 系统调用和库函数都是C函数的形式
- 库函数是通过调用系统调用实现的
- 应用程序既可以调用库函数，也可以调用系统调用

