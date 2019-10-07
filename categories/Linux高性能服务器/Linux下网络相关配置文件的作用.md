# Linux下网络相关配置文件的作用
/etc/resolv.conf　存放DNS服务器的ip地址，由内核程序写入不要更改。  
/etc/services    存放服务对应的端口编号  
/etc/hosts　存放主机名对应的ip地址，可以自己设定。需要通过主机查找ip地址时，将首先检查这个文件。未找到对应的IP地址时才会求助于dns服务。  
/etc/hosts.conf    设定找ip的先后顺序，默认是先hosts再dns。  
