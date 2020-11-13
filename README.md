# ELKStackDockerConfig

## 开发环境

Ubuntu 20.04

在本文中，Filebeat单独配置，Elasticsearch、Logstash和Kibana使用Docker方式配置；

## Filebeat配置与安装

安装：

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.10.0-linux-x86_64.tar.gz
tar xzvf filebeat-7.10.0-linux-x86_64.tar.gz
```

进入Filebeat安装目录，修改配置文件`filebeat.yml`，在这里可以修改Filebeat接收输入的方式，以及向哪里输出。在这里，我们设置Filebeat从UDP接受数据，输出到Logstash监听的5044端口，修改后的配置文件如下：

```
filebeat.inputs:
- type: udp
  enabled: true
  max_message_size: 10KiB
  host: "localhost:8888"

#----------------------------- Logstash output --------------------------------
output.logstash:
     hosts: ["127.0.0.1:5044"]

```

## Docker安装

用阿里云镜像安装：

```
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

为了使用docker-compose命令进行打包运行，需要下载该命令脚本：

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

然后为此脚本添加执行权限：

```
sudo chmod +x /usr/local/bin/docker-compose
```

## 配置ELK

### Logstash配置

在Logstash中，我们主要在`logstash.conf`中配置输入，过滤器和输出。输入规定了Logstash从哪里接受数据，过滤器定义了如何处理这些数据，比如字段的映射，输出规定了处理完的数据的流出方向。

在这里我们设置Logstash从filebeat中接受数据，输出到Elasticsearch中；过滤器是将从filebeat中接收到的数据进行映射，处理成json格式发送给输出，从filebeat中接受的数默认放在message域中，这里的映射是`"message" => "..."`，可以同时有多个输出，为了方便展示，这里也将处理过的数据输出到控制台。

```
input {
	beats {
		port => 5044
	}
}

filter {
  dissect {
    mapping => { "message" => "[%{ts} %{+ts}][%{levelname}][%{threadName}][%{name}:%{lineno}] %{message}" }
  }
}

output {
    stdout {
        codec => rubydebug
    }
    elasticsearch {
        hosts => "elasticsearch:9200"
				user => "elastic"
				password => "changeme"
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    }
}
```



### Elasticsearch配置

es中主要就是使用自带的X-pack插件进行安全性的配置，这样从Kibana中登录连接到es就需要输入密码；

### Kibana配置

在这个简单的例子中，只要设置一下index pattern即可。index pattern的原理是这样的，对于es的index，很多人习惯用项目名称+日期的方式建立index名称，比如`myproject-2020-11-11/id_01`，在Kibana中设置index pattern就可以将一类index都收集起来，比如我们设置index pattern为`myproject*`,这样每天es产生的index都可以被展示到Kibana中。

## 测试

clone本仓库到本地，在开启docker服务后，使用`docker-compose up`即可运行；

```
git clone ..
cd ..
docker-compose up
```

使用`filebeat`命令开启filebeat服务；

使用nc命令向8888端口发送UDP数据报：

```
nc -u localhost 8888
```

对于上面的filter方式，发送如下数据进行测试：

```
[2020-11-11 00:04:00,204][INFO][svc-metric-uploader-svc040][service.metric_uploader:88] This is the message we sent.
```

几乎同时可以在Logstash的终端输出中看到对应的json输出，因为我们之前配置过输出到终端中，然后再打开Kibana的Discover界面，就可以看到数据以及展示在时间序列图中；可供添加的字段在左侧，可以根据需要进行添加；