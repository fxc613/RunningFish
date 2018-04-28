# 服务器非ROOT用户定时任务

非root用户实现定时任务

说明：非root用户需要开通权限才能开启定时service服务crond。故需要root用户开启权限


# 1-	非root用户登陆 

# 2-	进入用户文件夹

cd /home/jsyfwork/
 
# 3-	创建定时任务文件夹

mkdir rsync_sh
 
# 4-	创建定时任务脚本

vim rsync_sh/rsync_bak.sh

#!/bin/sh

echo $(date +"%Y-%m-%d-%H:%M:%S")

rsync -auv --password-file=/etc/rsyncd.pass root@39.107.74.82::backup /fastdfs/storage/data/
 
# 5-	用户添加定时任务计划

crontab -e
 
15 19 * * * /home/jsyfwork/rsync_sh/rsync_bak.sh >> /home/jsyfwork/rsync_sh/rsync_bak.log 2>&1
 
# 6-	切换root用户

su root
 
# 7-	编辑sodu权限配置非root用户的crond权限

visudo
 
##注：在root配置下添加

jsyfwork ALL=(ALL)       NOPASSWD:/sbin/service
 
# 8-	配置定时任务脚本.sh文件的权限

chmod 744 /home/jsyfwork/rsync_sh/rsync_bak.sh
 
# 9-	为非root用户添加文件夹权限

chown -R jsyfwork:jsyfwork /fastdfs/
 
# 10-	转回非root用户

su jsyfwork
 
# 11-	重启crond定时服务

sudo service crond restart
 
