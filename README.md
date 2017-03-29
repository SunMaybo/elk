# ELK日志搜集平台解决方案
---
---
---
1. 硬件设备
2. 系统环境
3. elasticsearch 集群部署
4. kibana 部署
5. logstash 部署
6. filebeat 部署
7. 平台边界
8. 资料搜集

## 硬件设备
```
1. cpu: Intel(R) Pentium(R) CPU G3220 @ 3.00GHz 双核
2. memory： 8G
```
## 系统环境
```
1. centos 7
2. java8
```
### 系统环境准备（elasticsearch对系统资源要求）
 1. 修改进程所需要内存区域
   elasticsearch 5以上版本需要至少262144
   
   ```
   sysctl -w vm.max_map_count=262144
   ```
 2. 修改指定用户打开文件限制(elasticsearch对系统资源要求)

  ```
  sudo vim /etc/security/limits.conf
  redhat hard nofile 65536 
  redhat soft nofile 65536 
  
  - redhat 用户名
  - soft 指的是当前系统生效的设置值
  - hard 表明系统中所能设定的最大值
  - soft 的限制不能比har 限制高 
  - nofile 打开文件数目
  ```
  
## elasticsearch集群部署

### 所需机器

```
node95:  root 192.168.100.95 123456 (默认先启动为master节点)
node96:  root 192.168.100.96 123456
node97:  root 192.168.100.97 123456  

```
### 安装包下载
  1. 地址：<https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.2.tar.gz>
  2. 使用压缩包安装启动
  
### 修改配置文件(elasticsearch.yml)
 1. cluster.name ---> 指定集群名字，集群中所有节点集群名字必须相同。
 2. node.name ---> 节点名字
 3. network.host ---> 绑定的ip地址，该ip地址必须是本机器存在ip地址。
 4. http.port ---> HTTP协议端口
 5. transport.tcp.port 节点间交互的tcp端口
 6. discovery.zen.ping.unicast.hosts ---> 用于发现其他节点
 7. discovery.zen.minimum_master_nodes ---> 用于最小master节点数 计算方法：节点总数/2+1
 8. bootstrap.memory_lock ---> 是否锁定内存（boolean） 
 9. path.logs ---> 日志位置
 10. path.data ---> 数据位置
 11. 其它配置 ---> <http://www.cnblogs.com/hanyouchun/p/5163183.html> 
 
### 本次部署配置
 ```
 cluster.name: juxinli-test
 node.name: node95
 network.host: 192.168.100.95
 discovery.zen.ping.unicast.hosts: ["192.168.100.95:9300","192.168.100.96:9300","192.168.100.97:9300"]
 discovery.zen.minimum_master_nodes: 2
 ```
### jvm虚拟机设置(jvm.options)
 ```
  -Xms2g  ---> default
  -Xmx2g  ---> default
 ```
### 启动命令
 ```
 1. ./bin/elasticsearch
 2. ./bin/elasticsearch -d 后台启动
 3.  不能用root用户启动
 ``` 
### 查询API
#### 节点信息查询

##### path:[_nodes] 主要用于验证节点是否已经启动开来

  文档地址: <https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html>
  
  测试地址: <http://192.168.100.95:9200/_nodes>
#### 其它见API Document
<https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-get.html>
## kibana 部署
### 下载地址
<https://artifacts.elastic.co/downloads/kibana/kibana-5.2.2-linux-x86_64.tar.gz>
### 文件配置(kibana.yml)

```
1. server.port ---> 端口
2. server.hos ---> 地址
3. elasticsearch.url ---> elasticsearch 地址
```
### 启动
```
./bin/kibana
```
## logstash 部署
### 下载地址
<https://artifacts.elastic.co/downloads/logstash/logstash-5.2.2.tar.gz>
### 配置文件(logstash.conf)
输入，输出，过滤详细情况见文档:
<https://www.elastic.co/guide/en/logstash/current/configuration.html>
### 启动
```
./bin/logstash -f ./config/logstash.conf
```
## filebeat部署
###  下载地址
<https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.2-linux-x86_64.tar.gz>
### 配置文件(filebeat.yml)
```
- input_type: log
  paths:
    - ./root/CrawLog
output.logstash:
  hosts: ["192.168.100.96:5044"]
```
### 启动 
```
sudo ./filebeat -e -c filebeat.yml
```
## 平台边界
### 日志输入规范
```
1. 输出文件路径: /root/CrawLog
2. 文件名: ${机构ID_订单ID_任务ID}_Log.%d{yyyy-MM-dd-HH}.log
3. 日志格式: %date{yyyy-MM-dd HH:mm:ss.SSS} %level [%thread][%file:%line] - %msg%n
```
### logstash 输入，过滤，输出 规范
```
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
       "source" => "/var/log/juxinli/(?<institutionId>[a-zA-Z0-9]*)_(?<indentId>[a-zA-Z0-9]*)_(?<taskId>[a-zA-Z0-9]*)"
    }
  }
}

output {
  elasticsearch {
    hosts => ["192.168.100.95:9200","192.168.100.96:9200","192.168.100.97:9200"]
    manage_template => false
    index => "%{taskId}-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
  }
}

说明: 
     1. 输入HTTP端口 5044
     2. 通过任务ID创建索引
     3. 从路径中提取任务ID，机构ID，订单ID加入属性方便查询。
     4. 过滤通过正则

```

### 查询层规范
```
1. 可以很直观通过kibana查询
2. 查询接口可以通过阅读elasticsearch的官方文档
3. 查询接口也可以通过分析kibana查询的http请求
```
## 资料搜集

1. 配置索引模版 - <http://blog.csdn.net/asia_kobe/article/details/51192848>
2. grok 在线正则地址 - <http://grokdebug.herokuapp.com/


  
 
 
 
  

  


 