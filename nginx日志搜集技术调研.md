nginx 日志搜集解决方案

## 系统环境描述

``` 
java8 

logstash  --监控nginx日志文件
```    

## 技术描述

```
通过logstash监控nginx access.log 日志文件，根据过滤规则和输出适配进行日志转发。
```
   
## 提供方案

### nginx 日志 format 配置

```
log_format main '{"@timestamp":"$time_iso8601",'
'"host":"$server_addr",'
'"clientip":"$remote_addr",'
'"http_x_forwarded_for":"$http_x_forwarded_for",'
'"size":$body_bytes_sent,'
'"responsetime":$request_time,'
'"upstreamtime":"$upstream_response_time",'
'"upstreamhost":"$upstream_addr",'
'"http_host":"$host",'
'"request":"$request",'
'"url":"$uri",'
'"xff":"$http_x_forwarded_for",'
'"referer":"$http_referer",'
'"agent":"$http_user_agent",'
'"status":"$status"}';
```

### nginx 日志 字段描述

```
访问日志
        访问日志主要记录客户端访问Nginx的每一个请求，格式可以自定义。通过访问日志，你可以得到用户地域来源、跳转来源、使用终端、某个URL访问量等相关信息。Nginx中访问日志相关指令主要有两条：
        （1）log_format
        log_format用来设置日志格式，也就是日志文件中每条日志的格式，具体如下：
        log_format name(格式名称) type(格式样式)
        举例说明如下：
        log_format  main  '$server_name $remote_addr - $remote_user [$time_local] "$request" '
                        '$status $uptream_status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for" '
                        '$ssl_protocol $ssl_cipher $upstream_addr $request_time $upstream_response_time';
        上面部分为Nginx默认指定的格式样式，每个样式的含义如下：
        $server_name：虚拟主机名称。
        $remote_addr：远程客户端的IP地址。
        -：空白，用一个“-”占位符替代，历史原因导致还存在。
        $remote_user：远程客户端用户名称，用于记录浏览者进行身份验证时提供的名字，如登录百度的用户名scq2099yt，如果没有登录就是空白。
        [$time_local]：访问的时间与时区，比如18/Jul/2012:17:00:01 +0800，时间信息最后的"+0800"表示服务器所处时区位于UTC之后的8小时。
        $request：请求的URI和HTTP协议，这是整个PV日志记录中最有用的信息，记录服务器收到一个什么样的请求
        $status：记录请求返回的http状态码，比如成功是200。
        $uptream_status：upstream状态，比如成功是200.
        $body_bytes_sent：发送给客户端的文件主体内容的大小，比如899，可以将日志每条记录中的这个值累加起来以粗略估计服务器吞吐量。
        $http_referer：记录从哪个页面链接访问过来的。 
        $http_user_agent：客户端浏览器信息
        $http_x_forwarded_for：客户端的真实ip，通常web服务器放在反向代理的后面，这样就不能获取到客户的IP地址了，通过$remote_add拿到的IP地址是反向代理服务器的iP地址。反向代理服务器在转发请求的http头信息中，可以增加x_forwarded_for信息，用以记录原有客户端的IP地址和原来客户端的请求的服务器地址。
        $ssl_protocol：SSL协议版本，比如TLSv1。
        $ssl_cipher：交换数据中的算法，比如RC4-SHA。 
        $upstream_addr：upstream的地址，即真正提供服务的主机地址。 
        $request_time：整个请求的总时间。 
        $upstream_response_time：请求过程中，upstream的响应时间。
```
    
### logstash 输入输出规则

```
input{
        file{
                path => "/usr/local/nginx/logs/access.log"
                start_position => "beginning"
                codec => json 
        }
}

filter{
        mutate{

                split => [ "upstreamtime", "," ]
        }

        mutate {

                convert => [ "upstreamtime", "float" ]
        }
}

output{
        http {
                codec => json
                http_method => post
                content_type => "application/json"
                url => "http://192.168.100.203:7006/ITS_oa/log"
       }
}

```

### logstash 输入,过滤，输出规则配置

[官方指南](https://www.elastic.co/guide/en/logstash/current/index.html) 

### centos7 logstash 安装，启动命令

```
1. wget https://download.elastic.co/logstash/logstash/packages/centos/logstash-1.5.3-1.noarch.rpm
2. yum localinstall logstash-1.5.3-1.noarch.rpm
3. cd /etc/logstash/conf.d/
4. 01-logstash-initial.conf  创建文件，写入输入，过滤，输出规则
5. systemctl start logstash && systemctl status logstash
```

### 提供RestApi 接口

```
   1. http://192.168.100.203:7006/ITS_oa/log
   2. 参数为hash类型
```
