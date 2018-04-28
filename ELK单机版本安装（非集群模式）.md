# ELK单机版本安装（非集群模式）
访问地址：http://106.15.92.216:5601
选择左侧Dashboard

# ELK单机版本安装（非集群模式）
参考文章：https://www.cnblogs.com/huangxincheng/p/7918722.html
# 1准备工作
## 1.1-下载ElasticSearch + LogStash + Kibana软件
官方下载最新版 :https://www.elastic.co/cn/products。（本人存放地址：/usr/local/software）
## 1.2-创建ELK文件夹
cd /usr/local/

mkdir elk

进入elk文件夹

cd elk/

解压三个下载的三个软件

tar -zxvf /usr/local/software/elasticsearch-6.2.3.tar.gz

tar -zxvf /usr/local/software/logstash-6.2.3.tar.gz

tar -zxvf /usr/local/software/kibana-6.2.3-linux-x86_64.tar.gz

## 1.3-创建非root用户（elasticsearch不能由root用户启动）
useradd fuchao

passwd fuchao
## 1.4-将elasticsearch-6.2.3文件赋权限给非root用户（fuchao）
chown -R fuchao:fuchao /usr/local/elk/ elasticsearch -6.2.3/

## 1.5-减小ELK的运行内存
vim elasticsearch-6.2.3/config/jvm.options

-Xms128m

-Xmx128m

vim elasticsearch-6.2.3/config/elasticsearch.yml

network.host: 172.19.147.175

vim kibana-6.2.3-linux-x86_64/config/kibana.yml

server.host: "172.19.147.175"

elasticsearch.url: http://172.19.147.175:9200

vim logstash-6.2.3/config/jvm.options

-Xms128m

-Xmx128m

# 2-开始安装配置
## 2.1-安装配置elasticsearch
### 2.1.1-切换到非root用户（付超）
su fuchao
### 2.1.2-进入elasticsearch启动bin目录
cd /usr/local/elk/elasticsearch-6.2.3/bin/
### 2.1.3-启动elasticsearch服务

#./elasticsearch

使用nohup后台运行

nohup ./elasticsearch &

注意：

初次启动时容易报的错误总结参考：

https://www.cnblogs.com/sloveling/p/elasticsearch.html

max number of threads [2048] for user [lishang] likely too low, increase to

在root用户下查看所有用户的进程数

ulimit -a

修改进程数为2048

ulimit -u 4096

启动成功访问浏览器http://106.15.92.216:9200/提示elasticsearch版本信息

{

  "name" : "xISqvac",
  
  "cluster_name" : "elasticsearch",
  
  "cluster_uuid" : "6S3b_LHcToy7G8Hor1Pn0g",
  
  "version" : {
  
    "number" : "6.2.3",
    
    "build_hash" : "c59ff00",
    
    "build_date" : "2018-03-13T10:06:29.741383Z",
    
    "build_snapshot" : false,
    
    "lucene_version" : "7.2.1",
    
    "minimum_wire_compatibility_version" : "5.6.0",
    
    "minimum_index_compatibility_version" : "5.0.0"
    
  },
  
  "tagline" : "You Know, for Search"
  
}

## 2.2-安装配置kibana
### 2.2.1-root用户进入kibana的启动bin文件夹
cd /usr/local/elk/kibana-6.2.3-linux-x86_64/bin/
### 2.2.2启动kibana服务

#./ kibana

使用nohup后台运行

nohup ./kibana &

启动成功访问

http://106.15.92.216:5601/

出现kibana管理界面

## 2.3-安装配置logstash
### 2.3.1-root用户进入logstash的bin启动目录
cd /usr/local/elk/logstash-6.2.3/bin/
### 2.3.2-启动logstash服务
#### 2.3.2.1-初次进入bin目录可以创建conf文件（config.conf）
vim config.conf

input {

    file {
    
        type => "jsyf-service"
        
        path => "/usr/local/workspace/sys-service/logs/jsyf-sys-service.log"
        
        start_position => "beginning"
        
        sincedb_path => "/dev/null" #从头读
        
        codec => multiline {
        
            pattern => "^\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{1,3} "#已“2018-04-27 16:02:10,190 ”开头的为一段日志，不是的自动归为上一段
            
            negate => true
            
            what => "previous"
            
        }
        
        ignore_older=>0
        
    }
    
}

output {

  elasticsearch { hosts => "172.19.147.175" }
  
  stdout { codec => rubydebug }
  
}

#### 2.3.2.2-下载配置geoip服务文件
参考http://www.bkjia.com/Linuxjc/1231705.html

下载GeoLiteCity数据库

wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz

解压GeoLiteCity数据库

tar -zxvf GeoLite2-City.tar.gz

进入GeoLiteCity数据库

cd GeoLite2-City_20180403/

将GeoLiteCity数据库复制到配置的路径（database：/usr/local/elk/logstash-6.2.3/GeoLite2-City.mmdb）下

cp GeoLite2-City.mmdb /usr/local/elk/logstash-6.2.3/

#### 2.3.2.3-修改nginx配置

因为测试使用的是读取nginx的access.log日志文件，自己不会对日志文件的日志读取，选择投机取巧方法，将日志文件输出为json对象，让logstash的配置文件可以直接读取，在这里修改nginx的配置文件

vim /usr/local/nginx/conf/nginx.conf

在http{}里面最后添加转换操作，

日志保存路径到配置（path => "/var/log/nginx/access.log_json"）的路径下。

确保/var/log/目录下有nginx文件夹，没有就新建一个

log_format json '{"@timestamp":"$time_iso8601",'

               '"@version":"1",'
               
               '"host":"$server_addr",'
               
               '"client":"$remote_addr",'
               
               '"size":$body_bytes_sent,'
               
               '"responsetime":$request_time,'      #$request_time没有双引号表明该值为int类型
               
               '"domain":"$host",'
               
               '"url":"$uri",'
               
               '"status":"$status"}';
               
access_log /var/log/nginx/access.log_json json;

#### 2.3.2.4重启nginx

sudo killall nginx

/usr/local/nginx/sbin/nginx

####2.3.2.5启动logstash·

使用nohup后台运行

nohup ./logstash -f config.conf &


#3-其他问题

##3.1-nginx

nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

netstat -anp|grep 80

kill -9 ××


