# socket网络编程no route to host 问题
跑书上的socket编程简单发送接受的程序时出现了no route to host 问题，网上查找资料发现是vps的防火墙没关端口不开放，关掉防火墙之后就可以连通。  
vps : centos 6.10  
service iptables status 查看防火墙状态  
service iptables start    开启防火墙  
service iptables stop    临时关闭防火墙  
chkconfig iptables off    永久关闭防火墙  
