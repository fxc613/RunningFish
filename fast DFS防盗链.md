# fast DFS防盗链
Fast DFS防盗链：


原理：链接添加token验证，token由文件id（文件路径，不包含group）、http.anti_steal.secret_key（加密key）和时间戳（单位：秒）加密生成。

如：

原链接: http://106.14.187.127:8888/group1/M00/00/00/rBNJ1FpO5q-ATVUSAACvKsM0910306.png

加密后: http://106.14.187.127:8888/group1/M00/00/00/rBNJ1FpO5q-ATVUSAACvKsM0910306.png?token=59206be1d0b778d0c27da4f53fd11faf&ts=1515121530

此时链接内包含token和到期时间。

需要环境：已安装好fastDFS和nginx环境，且能通过原链接访问文件。

步骤：

设置服务器fast DFS配置

# 1-	修改fdfs里面的http.conf配置文件（路径根据个人服务器地址填写）

vim /etc/fdfs/http.conf 
 
修改内容：

http.anti_steal.check_token=true#设置为true表示开启token验证

http.anti_steal.token_ttl=900#设置token失效的时间单位为秒(s)

http.anti_steal.secret_key=FastDFS1234567890#密钥，跟客户端配置文件的fastdfs.http_secret_key保持一致

http.anti_steal.token_check_fail= /home/test.png#如果token检查失败，返回的页面

# 2-	修改完成，重新启动fastDFS服务

停止命令:

#停止tracker命令:

/etc/init.d/fdfs_trackerd stop 

#关闭storage命令:

/etc/init.d/fdfs_storaged stop 

#关闭nginx命令:

/usr/local/nginx/sbin/nginx -s stop

启动命令:

#启动tracker命令:

/etc/init.d/fdfs_trackerd start 

#启动storage命令:

/etc/init.d/fdfs_storaged start 

#查看fastdfs进程命令:

ps -el | grep fdfs 

#启动nginx命令:

/usr/local/nginx/sbin/nginx

JAVA客户端编写token生成方法

/**

 * @param remoteFilename 文件id（文件路径不包含group）
 
 * @param tokenTtl 有效时间，单位秒
 
 * @param secretKey 加密key,与服务器配置文件的fastdfs.http_secret_key保持一致
 
 * @Description: 通过文件id，有效时间，加密key获取fastDFS文件链接的token信息  <br/>
 
 * @return: void 此处为测试，不返回信息，只打印token和有效时间
 
 * @author: 付超
 
 * @createTime: 2018/1/5 11:30
 
 * @Version: V1.0.0
 
 */
 
public static void getFastDFSToken(String remoteFilename,int tokenTtl,String secretKey)

{

    try {
    
        //fastDFS的链接配置文件
        
        String configFileName = "D:/w-yingFeng/p5_server/shop/jsyf-sys/src/main/resources/fdfs_client.conf";
        
        ClientGlobal.init(configFileName);
        
        //到期时间，以秒为单位
        
        int ts = (int) (System.currentTimeMillis() / 1000);
        
        ts = ts + tokenTtl;
        
        //加密方式，fastdfs包里面的加密方式，，
        
        String token = ProtoCommon.getToken(remoteFilename, ts, secretKey);
        
        StringBuilder sb = new StringBuilder();
        
        sb.append("token=").append(token);
        
        sb.append("&ts=").append(ts);
        
        //打印token和到期时间
        
        System.out.println(sb.toString());
        
    } catch (IOException e) {
    
        e.printStackTrace();
        
    } catch (MyException e) {
    
        e.printStackTrace();
        
    } catch (NoSuchAlgorithmException e) {
    
        e.printStackTrace();
        
    }
}
  

public static void main(String[] args) {

    //文件路径"group1/M00/00/00/rBNJ1FpMfMqAFAuZAACvKsM0910151.png";需要去除group
    
    String remoteFilename = "M00/00/00/rBNJ1FpO5q-ATVUSAACvKsM0910306.png";
    
    //有效时间。单位为秒
    
    int tokenTtl = 900;
    
    //加密key,与服务器配置文件的fastdfs.http_secret_key保持一致
    
    String secretKey = "FastDFS1234567890";
    
    getFastDFSToken(remoteFilename,tokenTtl,secretKey);
    
}
 

FastDFS的链接配置文件内容：

connect_timeout = 200

network_timeout = 3000

charset = UTF-8

http.tracker_http_port = 8080

http.anti_steal.check_token=true

http.anti_steal.token_ttl=900

http.secret_key = FastDFS1234567890

#跟踪器：

tracker_server = 106.14.187.127:23000
 

