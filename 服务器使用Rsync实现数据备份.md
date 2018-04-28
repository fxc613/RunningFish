# 服务器使用Rsync实现数据备份

服务器环境：两台centOS7服务器

服务端服务器A（文件源服务器：106.15.92.216）

客户端服务器B（接收服务器：106.14.187.127）

目的：将服务端服务器A的/fastdfs/storage/data/目录里面的文件同步到客户端服务器B的/fastdfs/storage/data/目录下。

参考文章：

1- CentOS7下rsync实现服务器之间实时同步

http://blog.csdn.net/q290994/article/details/78079991?foxhandler=RssReadRenderProcessHandler

2-CentOS 7安装部署Rsync数据同步服务器

http://www.linuxidc.com/Linux/2017-06/144757.htm

3-CentOS 6.3 Rsync服务端与Debian 6.0.5 Rsync客户端实现数据同步

https://zm10.sm-tc.cn/?src=l4uLj8XQ0IiIiNGQjIaKkYialtGckJLQno2cl5aJmozQy8fMytGXi5KT&from=derive&depth=2&link_type=60&bu=web&v=1&uid=3f8ce86fc0e9196648a463cbf93c2a5f&hid=30a4f10eee32fc9d3f8046ef54ad5e53&restype=1&uc_param_str=dnntnwvepffrgibijbprsvdsdichei&query=rsync

# 服务端服务器A的rsync安装及配置

# 1-	安装rsync：
yum -y install  rsync
 
# 2-	因为rsync安装完成不会自动生成conf配置文件，所以需要自己创建rsyncd.conf文件；

vim /etc/rsyncd.conf
 
uid = root#设置进行数据传输时所使用的账户名称或ID号

gid = root#设置进行数据传输时所使用的组名称或GID号

port = 873#设置服务器监听的端口号，默认为873

use chroot = yes#设置user chroot为yes后，rsync会首先进行chroot设置，将根映射到path参数路径下，对客户端而言，系统的根就是path参数所指定的路径。但这样做需要root权限，并且在同步符号连接资料时仅会同步名称，而内容将不会同步。

read only = no#是否允许客户端上传数据，

list = no#客户端请求显示模块列表时，本模块名称是否显示，

max connections = 4#设置并发连接数，0代表无限制。超出并发数后，如果依然有客户端连接请求，则将会收到稍后重试的提示消息

pid file = /var/run/rsyncd.pid#设置Rsync进程号保存文件名称

timeout = 900#超时时间

motd file  = /etc/rsyncd/rsyncd.motd#设置服务器信息提示文件名称，在该文件中编写提示信息

log file = /var/log/rsyncd.log#设置日志文件名称，

lock file = /var/run/rsyncd.lock#设置锁文件名称 

[backup]#模块，Rsync通过模块定义同步的目录，模块以[name]的形式定义，这与Samba定义共享目录是一样的效果。在Rsync中也可以定义多个模块

        comment = this is module for backup#comment定义注释说明字串 
        
        path = /fastdfs/storage/data/#同步目录的真实路径通过path指定 
        
        ignore errors#忽略一些IO错误 
        
        auth user = root#设置允许连接服务器的账户，账户可以是系统中不存在的用户 
        
        	  secrets file = /etc/rsyncd.pass#设置密码验证文件名称，注意该文件的权限要求为只读，建议权限为600，仅在设置auth users参数后有效

uid = root

gid = root

port = 873

use chroot = yes

read only = no

list = no

max connections = 4

pid file = /var/run/rsyncd.pid

timeout = 900

motd file  = /etc/rsyncd/rsyncd.motd

log file = /var/log/rsyncd.log

lock file = /var/run/rsyncd.lock

[backup]

        comment = this is module for backup
        
        path = /fastdfs/storage/data/
        
        ignore errors
        
        auth user = root
        
        secrets file = /etc/rsyncd.pass

# 3-	生成并编写用户密码文件

echo "root:123" > /etc/rsyncd.pass
 
此时会自动生成密码文件rsyncd.pass

vim /etc/rsyncd.pass
 
 
会看到账号为root，密码是123

注：该文件可以设置多个账号和密码，每行一个用户名:密码

# 4-	修改密码文件rsyncd.pass的权限

chmod 600 /etc/rsyncd.pass

注：600-设置文件所有者读取、写入权限

# 5-	启动rsyncd服务

service rsyncd start 
 
# 6-	查看进程占用端口（rsyncd默认端口：873）

netstat -tunlp

可以看到873端口已经启用,表示启动成功

此时如果rsyncd没有启用，则需要检查防火墙端口是否开启873端口

## 6.1-查看防火墙配置

vim /etc/sysconfig/iptables
 
我们这里已经添加了开放873端口

如果没有则添加开放指令

-A INPUT -p tcp -m state --state NEW -m tcp --dport 873 -j ACCEPT

保存防火墙配置，让后使其生效

service iptables restart
 
# 客户端服务器B的rsync安装与配置
# 1-	安装rsync:

yum -y install  rsync
 
# 2-	设置访问账号及密码

echo "root:123" > /etc/rsyncd.pass
 
注：这里的账号及密码需要是服务端服务器A第三步设置的账号与密码
# 3-	设置密码文件权限

chmod 600 /etc/rsyncd.pass
 
# 4-	执行指令（将服务端服务器A的backup模块下配置/fastdfs/storage/data/目录的文件同步到客户端服务器B的/fastdfs/storage/data/目录下）

rsync -auv --password-file=/etc/rsyncd.pass root@106.15.92.216::backup /fastdfs/storage/data/

连接失败：

检查服务端服务器A的防火墙873端口及添加安全组规则是否有873端口
 
添加873安全组规则完成后，再执行指令提示同步成功
 
