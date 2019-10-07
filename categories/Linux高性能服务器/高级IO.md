# 高级IO
### pipe函数
```
int pip(int fd[2]);
```
fd[0]，和fd[1]构成管道的两端。fd[0]只能读而fd[1]只能写。默认情况下这一对描述符都是**阻塞**的。  
socket API中有一个sockpair函数，能够方便快捷的创建双向管道。
```
int socketpair(int domain, int type, int protocol, int fd[2]);
```
前三个参数与socket中的相同，但domain只能是AF_UNIX只能在本地使用这个双向管道。创建成功后这对描述符即是既可读又可写。  
```
int dup(int file_descriptor) // 将新描述符和file_descriptor描述符指向同一个文件、管道、socket
// 使用iovec读写非连续的内存
ssize_t readv(int fd, const struct iovec* vector, int count);
sszie_t writev(int fd, const struct iovec * vector, int count);
```

```
ssize_t sendfile(int out_fd, int in_fd, off_t* offset, size_t count);
```
sendfile函数在两个描述符之间传递数据，操作完全在内核中完成，效率很好称为零拷贝。  
**注意**，in_fd函数必须指向真实的文件而不能是socket或者管道。而out_fd必须是一个socket。sendfile函数是专门为**网络上传输文件**而设计的。  
```
void *mmap(void *start, size_t length, int port, int flags, int fd, off_t offset);
int munmap(void *start, size_t length);
```
mmap函数申请一段内存，可以用于进程间通讯的共享内存，也可以将文件映射到其中。port用来设置段的访问权限。其中flag控制内存段内容被修改后程序的行为。  

```
sszie_t splice(int fd_in, loff_t* off_in, int fd_out, loff_t* off_out, size_t len, unsigned int flags);
```
splice函数用于在两个文件描述符之间移动数据，是**零拷贝操作**。fd_in和fd_out必须至少有一个是**管道文件描述符**。

```
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```
其含义和splice函数相同，也是个**零拷贝函数**。用于在两个管道描述符之间复制数据，fd_in和fd_out都必须是管道文件描述符。  

### I/O模型
#### 阻塞与非阻塞
socket在创建时可以设置SOCK_NONBLOCK让socket变为非阻塞的。  
阻塞的I/O系统调用可能应为无法立即完成而被操作系统挂起，直到等到事件发生为止。socket API中可能被阻塞的有accept、send、recv、connect。  
非阻塞IO的系统调用总是立即返回，不论事件是否已经发生。需要根据errno来具体看为发生成功的具体情况。非阻塞IO通常要和其他的IO通知机制一起使用，如IO复用和SIGIO信号。  

#### 同步与异步
同步IO模型，IO的读写操作都是在IO事件发生之后由应用程序来完成的。阻塞IO、IO复用和信号驱动IO都是同步IO模型。  
对于异步IO，用户可以直接对IO执行读写操作，这些操作告诉内核用户读写缓冲区的位置以及IO操作完成之后内核通知应用程序的方式。异步IO的读写操作总是立即返回而不论IO是否阻塞，真正的读写操作由内核接管。  
**同步IO向应用程序通知的是IO就绪事件，而异步IO向应用程序通知的是IO完成事件。**
