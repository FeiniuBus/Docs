## 1.环境介绍
服务器环境
- Server AWS Ubuntu Trusty 14.04 (LTS) x64
- elk 组件 
  - logstash 5.0.0
  - kibana 5.0.0
  - elasticsearch 5.0.0
  - redis 

## 2.架构说明
- client：程序通过udp端口发送json 格式的日志消息到  
- server
  - logstash shipper 收集日志
  - redis 缓存
  - elasticsearch 处理、保存日志
  - kibana 前端展示

## 3.组件安装参数配置
>###### java 环境配置
>###### logstash安装配置
>###### elasticsearch安装配置
>###### kibana 安装配置
>###### redis 安装
<p></p>

### java 环境配置
java 下载 [https://www.java.com/zh_CN/download/manual.jsp](https://www.java.com/zh_CN/download/manual.jsp)  
```bash
weget http://sdlc-esd.oracle.com/ESD6/JSCDL/jdk/8u111-b14/jre-8u111-linux-x64.tar.gz?GroupName=JSC&FilePath=/ESD6/JSCDL/jdk/8u111-b14/jre-8u111-linux-x64.tar.gz&BHost=javadl.sun.com&File=jre-8u111-linux-x64.tar.gz&AuthParam=1477997592_8023dba3df857730418d4bd15a12ec0b&ext=.gz`  
sudo mkdir -p /usr/local/java/    
sudo tar -zxvf jre-8u111-linux-x64.tar.gz -C /usr/local/java/
sudo cp /etc/environment /etc/environment.bak
sudo vim /etc/environment
```
修改PATH参数，添加java路径 :/usr/local/java/jre1.8.0_111/bin  
```bash
java -version
```
>验证 java 环境  
>ubuntu@ubuntu:~$ java -version  
>java version "1.8.0_111"  
>Java(TM) SE Runtime Environment (build 1.8.0_111-b14)  
>Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)  



### 下载elk套件
官网地址：[https://www.elastic.co/products](https://www.elastic.co/products)</p>
所有组件下载5.0.0版本，tar.gz 压缩包

### logstash安装配置
```bash
sudo tar -zxvf logstash-5.0.0.tar.gz -C /usr/local/
sudo chown -R ubuntu.ubuntu /usr/local/logstash-5.0.0/
```
### elasticsearch安装配置
```bash
sudo tar -zxvf elasticsearch-5.0.0.tar.gz -C /usr/local/
sudo chown -R ubuntu.ubuntu /usr/local/elasticsearch-5.0.0/
```
### kibana 安装配置
```bash
tar -zxvf kibana-5.0.0-linux-x86_64.tar.gz -C /usr/local/
sudo chown -R ubuntu.ubuntu /usr/local/kibana-5.0.0-linux-x86_64/
vim /usr/local/kibana-5.0.0-linux-x86_64/config/kibana.yml
```
找到server.hsot 参数 作如下修改
```bash
#server.host: "localhost"
server.host: "0.0.0.0"
```
### redis 安装
```bash
sudo apt-get install redis-server
```

# 4.服务器logstash端配置
>###### shipper配置文件  
>###### index配置文件  

### shipper配置文件
```bash
cd /usr/local/logstash-5.0.0/
mkdir conf
cd conf
vim ./shipper.conf
```
添加以下配置  
```ruby
input {  
        udp {
                host =>  "0.0.0.0"
                port => "8899"
        }
}

output {
        stdout {
        }
    redis {
        host => "127.0.0.1"
        port => "6379"
        data_type => "channel"
        key => "yourkeyname"
    }
}
```
### index配置文件

```bash
cd /usr/local/logstash-5.0.0/conf
vim index.conf
```
添加配置
```ruby
input {
        redis {
                host => "127.0.0.1"
                data_type => "channel"
                key => "yourkeyname"
        }
}

filter {
        json { source => "message" }
}

output {
        stdout { }
        elasticsearch {
                hosts => ["127.0.0.1:9200"]
                index => "testlog"
                codec => "json"
        }
}
```

# 5.测试运行
>###### 启动组件
>###### 通过supervisor启动

### 启动组件
```bash
/usr/local/elasticsearch-5.0.0/bin/elasticsearch
/usr/local/kibana-5.0.0-linux-x86_64/bin/kibana
/usr/local/logstash-5.0.0/bin/logstash -f /usr/local/logstash-5.0.0/conf/index.conf
/usr/local/logstash-5.0.0/bin/logstash -f /usr/local/logstash-5.0.0/conf/shipper.conf
```
>正式环境不推荐这种启动

### 通过supervisor启动
```bash
sudo apt-get install supervisor -y
sudo service supervisor start
sudo vim /etc/supervisor/conf.d/elk.conf
```
添加以下配置
```conf
[program:es]
user=ubuntu
environment=LS_HEAP_SIZE=5000m
directory=/usr/local/elasticsearch-5.0.0/
command=/usr/local/elasticsearch-5.0.0/bin/elasticsearch

[program:kibana]
user=ubuntu
environment=LS_HEAP_SIZE=5000m
directory=/usr/local/kibana-5.0.0-linux-x86_64/
command=/usr/local/kibana-5.0.0-linux-x86_64/bin/kibana

[program:ls-shipper]
user=ubuntu
environment=LS_HEAP_SIZE=5000m
directory=/usr/local/logstash-5.0.0/
command=/usr/local/logstash-5.0.0/bin/logstash -f /usr/local/logstash-5.0.0/conf/shipper.conf

[program:ls-index]
user=ubuntu
environment=LS_HEAP_SIZE=5000m
directory=/usr/local/logstash-5.0.0/
command=/usr/local/logstash-5.0.0/bin/logstash -f /usr/local/logstash-5.0.0/conf/index.conf
```
启动supervisor 
```bash
sudo supervisorctl start all
```
执行后会有报错  
ls-shipper: ERROR (abnormal termination)  
ls-index: ERROR (abnormal termination)  
es: ERROR (abnormal termination)  
>解决办法：sudo ln -sv /usr/local/java/jre1.8.0_111/bin/java /usr/bin/java

# 6.通过kibana进行日志管理
>###### 测试数据发送
>###### index 管理
>###### 图表管理



### 测试数据发送
下载udp客户端工具，向logstash 发送测试数据
```json
{
    "class":"ERROR",
    "logger_name":"testlogger",
    "application":"udpsender",
    "message":"high Hkaos three",
    "addUser":"tester",
    "addUserName":"testName",
    "objName":"objName",
    "objId":"objId",
    "hash":-1,
    "method":"Main",
    "timestamp":"0001-01-01T00:00:00"
}
```
### index 管理
在上文 logstash index配置文件中的output 配置了index => testlog  
访问 http://yourserverip:5601/  
在Management --> Index Patterns --> add new 添加testlog  
![创建index](http://upload-images.jianshu.io/upload_images/3087483-a2ba02b1835663d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果没有识别出来的话说明 logstash 和 elasticsearch 的通讯异常，或者启动有问题，查看日志进行排查  

到此本次安装基本完成，kibana 页面上的数据管理不难理解，本文没有进行说明  

## 参考文档
[中文参考文档](http://udn.yyuap.com/doc/logstash-best-practice-cn/index.html)  
[logstash官方手册](https://www.elastic.co/guide/en/logstash/current/index.html)
