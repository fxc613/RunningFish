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
        
        start_position => "end"
        
        sincedb_path => "/dev/null" #从头读
        
        codec => multiline {
        
            pattern => "^\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{1,3} "#已“2018-04-27 16:02:10,190 ”开头的为一段日志，不是的自动归为上一段
            
            negate => true
            
            what => "previous"
            
            auto_flush_interval => 2
            
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

#### 2.3.2.5启动logstash·

使用nohup后台运行

nohup ./logstash -f config.conf &


# 3-其他问题

## 3.1-nginx

nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

netstat -anp|grep 80

kill -9 ××

# 4-其他服务器配置logstash

# 4.1-其他Linux服务器的logstash

安装同当前服务器，

配置文件conf.conf内容

input {

    file {
    
        type => "jsyf-service"
        path => "/usr/local/workspace/sys-service/logs/jsyf-sys-service.log"
        
        start_position => " end "
        
        sincedb_path => "/dev/null" #从头读
        
        codec => multiline {
        
            pattern => "^\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{1,3} "
            
            negate => true
            
            what => "previous"
            
            auto_flush_interval => 2
            
        }
        
        ignore_older=>0
        
    }
    
}

output {

  elasticsearch { hosts => "106.15.92.216" }
  
  stdout { codec => rubydebug }
  
}

启动

## 4.2-Windows服务器的logstash

官网https://www.elastic.co/cn/products下载windows的安装包（.zip）

最好解压在没有中文和特殊字符的文件目录

在bin目录下创建conf.conf配置文件。内容为

input {

    file {
    
        type => "jsyf-service"
        
        path => "D:\w-yingFeng\p5_server\logs\jsyf-sys-service.log"
        
        start_position => "end"
        
        sincedb_path => "E:\logstash\logstash-6.2.4\bin\null" #从头读
        
        codec => multiline {
        
            pattern => "^\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{1,3} "
            
            negate => true
            
			      what => "previous"
            
			      auto_flush_interval => 2
            
        }
        
        ignore_older=>0
        
    }
    
}

output {

  elasticsearch { hosts => "106.15.92.216" }
  
  stdout { codec => rubydebug }
  
}

然后创建启动文件run.bat。内容是

logstash.bat -f conf.conf

然后点击run.bat启动logstash

# 附：logstash配置说明

参考文档：

http://www.voidcn.com/article/p-xboozfbl-bov.html

https://blog.csdn.net/xfg0218/article/details/52980726

#整个配置文件分为三部分：input,filter,output

#参考这里的介绍 https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html

input {

  #file可以多次使用，也可以只写一个file而设置它的path属性配置多个文件实现多文件监控
  
  file {
  
    #type是给结果增加了一个属性叫type值为"<xxx>"的条目。这里的type，对应了ES中index中的type，即如果输入ES时，没有指定type，那么这里的type将作为ES中index的type。
    
    type => "apache-access" 
    
    path => "/apphome/ptc/Windchill_10.0/Apache/logs/access_log*"
    
    #start_position可以设置为beginning或者end，beginning表示从头开始读取文件，end表示读取最新的，这个也要和ignore_older一起使用。
    
    start_position => beginning
    
    #sincedb_path表示文件读取进度的记录，每行表示一个文件，每行有两个数字，第一个表示文件的inode，第二个表示文件读取到的位置（byteoffset）。默认为$HOME/.sincedb*
    
    sincedb_path => "/opt/logstash-2.3.1/sincedb_path/access_progress"
    
    #ignore_older表示了针对多久的文件进行监控，默认一天，单位为秒，可以自己定制，比如默认只读取一天内被修改的文件。
    
    ignore_older => 604800
    
    #add_field增加属性。这里使用了${HOSTNAME}，即本机的环境变量，如果要使用本机的环境变量，那么需要在启动命令上加--alow-env。
    
    add_field => {"log_hostname"=>"${HOSTNAME}"}
    
    #这个值默认是\n 换行符，如果设置为空""，那么后果是每个字符代表一个event
    
    delimiter => ""
    
    #这个表示关闭超过（默认）3600秒后追踪文件。这个对于multiline来说特别有用。... 这个参数和logstash对文件的读取方式有关，两种方式read tail，如果是read
    
    close_older => 3600
    
    coodec => multiline {
    
      pattern => "^\s"
      
      #这个negate是否定的意思，意思跟pattern相反，也就是不满足patter的意思。
      
      #negate => ""
      
      #what有两个值可选 previous和next，举例说明，java的异常从第二行以空格开始，这里就可以pattern匹配空格开始，what设置为previous意思是空格开头这行跟上一行属于同一event。另一个例子，有时候一条命令太长，当以\结尾时表示这行属于跟下一行属于同一event，这时需要使用negate=>true，what=>'next'。
      
      what => "previous"
      
      auto_flush_interval => 60
      
    }
    
  }
  
  file { 
  
    type => "methodserver-log" 
    
    path => "/apphome/ptc/Windchill_10.0/Windchill/logs/MethodServer-1604221021-32380.log" 
    
    start_position => beginning 
    
    sincedb_path => "/opt/logstash-2.3.1/sincedb_path/methodserver_process"
    
    #ignore_older => 604800
    
  }
  
}

filter{

  #执行ruby程序，下面例子是将日期转化为字符串赋予daytag
  
  ruby {
  
    code => "event['daytag'] = event.timestamp.time.localtime.strftime('%Y-%m-%d')"
    
  }
  
  #if [path] =~ "access" {} else if [path] =~ "methodserver" {} else if [path] =~ "servermanager" {} else {} 注意语句结构
  
  if [path] =~ "MethodServer" { #z这里的=~是匹配正则表达式
  
    grok {
    
      patterns_dir => ["/opt/logstash-2.3.1/patterns"] #自定义正则匹配
      
      #Tue 4/12/16 14:24:17: TP-Processor2: hirecode---->77LS
      
      match => { "message" => "%{DAY:log_weekday} %{DATE_US:log_date} %{TIME:log_time}: %{GREEDYDATA:log_data}"}
      
    }
    
    #mutage是做转换用的
    
    mutate { 
    
      replace => { "type" => "apache" } #替换属性值
      
      convert => { #类型转换
      
        "bytes" => "integer" #例如还有float
        
        "duration" => "integer"
        
        "state" => "integer"
        
      }
    #date主要是用来处理文件内容中的日期的。内容中读取的是字符串，通过date将它转换为@timestamp。参考https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html#plugins-filters-date-match
    
    #date {
    
    #match => [ "logTime" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    
    #}
    
  }else if [type] in ['tbg_qas','mbg_pre'] { # if ... else if ... else if ... else结构
  
  }else {
  
    drop{} # 将event丢弃
    
  }
  
}

output {

  stdout{ codec=>rubydebug} # 直接输出，调试用起来方便
  
  #输出到redis
  
  redis {
  
    host => '10.120.20.208'
    
    data_type => 'list'
    
    key => '10.99.201.34:access_log_2016-04'
    
  }
  #输出到ES
  
  elasticsearch {
  
    hosts =>"192.168.0.15:9200"
    
    index => "%{sysid}_%{type}"
    
    document_type => "%{daytag}"
    
  }
  
}


