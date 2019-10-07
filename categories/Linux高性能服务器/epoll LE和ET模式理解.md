# epoll LE和ET模式理解
## epoll的两种模式LT和ET
二者的差异在于level-trigger模式下只要某个socket处于readable/writable状态，无论什么时候进行epoll_wait都会返回该socket；而edge-trigger模式下只有某个socket从unreadable变为readable或从unwritable变为writable时，epoll_wait才会返回该socket。如下两个示意图:

从socket读数据:

![](_v_images/20190509171611970_1527337491.png)


往socket写数据
![](_v_images/20190509171626984_590034225.png)


所以，在epoll的ET模式下，正确的读写方式为:
读：只要可读，就一直读，直到返回0(#add 读空)，或者 errno = EAGAIN
写:只要可写，就一直写，直到数据发送完（#add 写满），或者 errno = EAGAIN

## ET模式下为什么要设置在非阻塞模式下工作
ET 模式下需要一直读一直写将缓存写满或者读空，直到errno = EAGIAN。在最后一次读写时由于读空或者写满，一定会产生阻塞造成别的fd饥饿。  

**从本质上讲：与LT相比，ET模型是通过减少系统调用来达到提高并行效率的。**