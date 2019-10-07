# IO多路转接
  
  阻塞与非阻塞的区别：  
  读：没有数据到达的时候，是否立刻返回。  
  写：阻塞情况：一直阻塞直到所有的数据都写完。非阻塞情况：有多少写多少。  
  
  问题：一个进程负责向两个文件读写，如何设计IO读写方式使得读写的更高效？  
### 1.使用子进程控制另一条读写通道
以读为例，一种思路是使用阻塞读read，通过进程间的信号传递告知对方进程我这里已经读完了，你可以使用IO进程读写。第二种思路是使用轮询，轮询read看是否有数据需要读，有则读无则略过。但是这种不推荐使用，因为很浪费CPU而且轮询时间也很难确定。  

### 2.异步IO
进程告诉内核，当描述符准备好可以进行IO时，用一个信号通知它。存在一些细节问题**待补充 14.5**  
### 3.IO多路转接**
构造一张需要操作的描述符列表，然后调用一个函数，直到这些描述符中的一个已经准备好进行IO时，该函数才返回。  
#### select函数
```
#inlcude <sys/select.h>

int select(int maxfdp1, fd_set *restrict readfds,
           fd_set *restrict writefds, fd_set *restrict exceptfds,
           struct timeval *restrict tvptr);
```
**返回值说明**  
-1表示出错。0表示没有一个描述符准备好。一个正值表示三个fd_set中准备好的描述符数目*之和*  
**参数说明**  
*tvptr*: tvptr == NULL 则永远等待除非有中断信号。结构中的tv_sec tv_usec设置需要等待的秒和微妙。  
*readfds*、*writefds*、*exceptset*：分别指向可读、可写和异常等事件对应的描述符集合。  
fd_set 类型可以看成是指定描述符是否加入检测列表，0为不加入，1为加入。  
*maxfdp1* 的值是最大文件描述符编号+1，因为描述符是从0开始的。  
与select函数相关的宏  
```
#include <sys/select.h>   
int FD_ZERO(int fd, fd_set *fdset);   //一个 fd_set类型变量的所有位都设为 0
int FD_CLR(int fd, fd_set *fdset);  //清除某个位时可以使用
int FD_SET(int fd, fd_set *fd_set);   //设置变量的某个位置位
int FD_ISSET(int fd, fd_set *fdset); //测试某个位是否被置位
``` 

#### select的用法细节（待补充）

#### select 在网络编程中的应用（待补充）

#### select的实现细节？（待补充）