# IO复用--epoll系列系统调用
Linux下实现I/O复用的系统调用主要有select、poll、epoll。虽然IO复用能同时监听多个文件描述符，但他本身还是**阻塞**的，当多个文件描述符同时就绪时，程序只能依次处理其中每个文件描述符，这样服务器就是串行的。如果要实现并发，只能使用多进程或多线程编程。  
## epoll 系列API
epoll时linux特有IO复用函数，在实现上与select、poll有很大差异。表现在
* epoll使用一组函数来完成任务，而不是单个函数
* epoll把用户关系你的文件描述符都放在内核的一个时间表中，无需像select和poll那样每次调用都要重复传入文件描述符集或事件集。  

使用epoll_create创建事件表  
```
// 返回文件描述符
int epoll_create(int size);
```
操作epoll内核事件表  
```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
fd是要操作的文件描述符，op指定操作类型，event参数指定事件。  
epoll_event的结构体如下
```
struct epoll_event{
    __uni32_t events; // 指定epoll事件
    epoll_data_t data; //用户数据
};
struct epoll_data{
    void *ptr; // 自定义指定与fd相关的用户数据
    int fd;  // 指定事件从属的目标文件符
    unit32_t u32;
    unit64_t u64;
} epoll_data_t;
```

epoll系列系统调用的主要接口是epoll_wait函数，等待一组文件描述符上的事件  
```
int epoll_wait(int epfd, struct epoll_event* events, int maxevents,
               int timeout);
```
如果检测到事件，就将所有的就绪事件从内核事件表中复制到events指向的数组中。  
与poll和select不同的是，这个函数只用于检测就绪事件而不用传入用户注册事件，提高应用程序索引文件描述符的效率。  

## LT和ET模式
epoll对文件描述符的操作有两种模式：LT（电平触发）和ET（边沿触发）。LT模式是默认的工作模式，此时相当于一个高效率的poll。往epoll内核事件表中注册一个文件描述法上的EPOLLET事件会以ET模式来操作文件描述符， ET是EPOLL的高效率工作模式。  
LT模式下，epoll_wait检测到其上有事件发生将此事件通知给应用程序，应用可以不立即处理事件，但下次调用epoll_wait还会再次向应用通告此事件。  
ET模式，检测到通知应用程序后必须立即处理，下次再调用不会再通知应用。ET模式降低了同一个epoll事件被重复触发的次数，效率要高于LT模式。  
  
  
在设计LT和ET模式的读写函数时要注意，由于ET模式不会重复通告一个事件，所以ET模式要在一次通告内全部完成操作。  
```
// lt模式的工作流程
void lt( epoll_event* events, int number, int epollfd, int listenfd )
{
    char buf[ BUFFER_SIZE ];
    for ( int i = 0; i < number; i++ )
    {
        int sockfd = events[i].data.fd;
        if ( sockfd == listenfd )
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof( client_address );
            int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
            addfd( epollfd, connfd, false ); // connfd禁ET模式
        }
        // 是一个读请求事件
        else if ( events[i].events & EPOLLIN )
        {
            printf( "event trigger once\n" );
            memset( buf, '\0', BUFFER_SIZE );
            int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );
            if( ret <= 0 )
            {
                close( sockfd );
                continue;
            }
            printf( "get %d bytes of content: %s\n", ret, buf );
        }
        else
        {
            printf( "something else happened \n" );
        }
    }
}
// ET模式的工作流程
void et( epoll_event* events, int number, int epollfd, int listenfd )
{
    char buf[ BUFFER_SIZE ];
    for ( int i = 0; i < number; i++ )
    {
        int sockfd = events[i].data.fd;
        if ( sockfd == listenfd )
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof( client_address );
            int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
            addfd( epollfd, connfd, true );
        }
        else if ( events[i].events & EPOLLIN )
        {
            printf( "event trigger once\n" );
            while( 1 ) // 因为ET模式不能被重复触发，所以必须循环读把socket缓存中的数据全部读干净
            {
                memset( buf, '\0', BUFFER_SIZE );
                int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );

                if( ret < 0 )
                {
                    // 对于非阻塞IO，下列条件表示数据已经全部读取完毕
                    if( ( errno == EAGAIN ) || ( errno == EWOULDBLOCK ) )
                    {
                        printf( "read later\n" );
                        break;
                    }
                    close( sockfd );
                    break;
                }
                else if( ret == 0 )
                {
                    close( sockfd );
                }
                else
                {
                    printf( "get %d bytes of content: %s\n", ret, buf );
                }
            }
        }
        else
        {
            printf( "something else happened \n" );
        }
    }
}

```

使用EPOLLONESHOT事件,可以让在多线程下避免出现两个线程同事操作一个sokcet的情况.在ET模式中，重新注册事件可以让内核重新通知应用程序描述符被唤醒。  
