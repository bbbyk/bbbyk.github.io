# Linux网络编程基础API
## 主机字节序和网络字节序  
不同的主机，甚至不同的进程之间（ＪＡＶＡ虚拟机是大端）其字节序都可能不一样，网络字节序的规定是大端的，任何格式化的数据通过网络传输时都应该使用转换函数来进行转换。htonl、htons、ntohl、ntohs。  

## 网络编程一般套路总结
服务端创建socket的一般方式
```
int main(int argc, char *argv[])
{
    if (argc <= 2){
        printf("need more args\n");
    }
    char *ip = argv[1];
    int port = atoi(argv[2]);

    // 填充服务端地址信息　struct addrr_in
    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(struct sockaddr_in)); // 0初始化addr
    server_addr.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_addr.sin_addr);
    server_addr.sin_port = htons(port);

    //  创建套接字
    int sockfd = socket(PF_INET, SOCK_STREAM, 0); // protocol一般都是０
    
    return 0;
}
```
### 命名socket
将socket与sokcet地址进行绑定称为给socket命名，使用bind()函数来实现。  
在服务器程序中同城要命名socket，服务器端口是公开约定的，这样客户端才知道怎么去连接它。客户端不需要命名sokcet，系统自动分配临时端口就ok。  

### 监听socket
```
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```
内核监听sockfd指向的socket，backlog指所有处于完全连接状态的socket上限（监听队列的最大长度），backlog的典型参数值是５。使用netstat -nt 查看发现处于ESTABLISHED比backlog要略多。  

### 接受连接
```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)
```
从listen监听队列中接受一个连接。sockfd是执行过listen系统监听的socket，函数返回一个被接受的sockfd。**accept的作用只是从监听队列中取出连接，不论连接处于何种状态，也不关心网络的变化。**  

### 发起连接
```
int connect( int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen);
```
客户端需要使用这个函数主动与服务器建立连接。sockfd是系统调用返回的。
### 关闭连接
```
int close(int fd);
```
close只是将sockfd的引用计数－１，注意在多进程时，fork之后子进程会让sockfd引用+1。所以我们要在父子进程中都close。强行终止sockfd使用shuodown函数。  

### 数据读写

TCP读写
```
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(itn sockfd, const void *buf, size_t len, int flags);
```
注意recv可能需要多次调用才能读取到完整的数据。flags提供额外的控制，如发送紧急数据MSG_OOB。  
### UDP数据读写
```
ssize_t recvform(int sockfd, void *buf, size_t len, int flags, struct sockaddr* src_addr, socklen_t* addrlen);
ssize_t sendto(int sockfd, const void* buf, size_t len, int flags, const struct sockaddr* dest_addr, socklen_t addrlen);
```
src_addr 和dest_addr分别指发送端的socket地址和接受端的socket地址。  
