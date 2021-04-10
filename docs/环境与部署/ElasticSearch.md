## 安装

### 安装ElasticSearch

> 拉取镜像

```bash
docker pull elasticsearch:7.9.3
```

>  启动容器

同时挂载目录（包括配置文件和data）（挂载出来的位置自己定义）

```bash
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d \
-v /home/es/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /home/es/data:/usr/share/elasticsearch/data elasticsearch:7.9.3
```

注意这里还设置了`JVM的内存`大小，默认为2G，有点大，很可能会因为内存不够而无法正常启动。可以像我这里改为256m或者其他值。

> 可能出现的错误

查看容器日志

```bash
docker logs elasticsearch
```

如果出现以下错误

```bash
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

则要修改服务器配置

```bash
vim /etc/sysctl.conf
```

添加这行

```bash
vm.max_map_count=262144
```

立即生效, 执行：

```bash
/sbin/sysctl -p
```

对挂载的宿主机data目录可能出现权限不足问题

```bash
chmod 777 [宿主机data目录]
```

> 配置跨域

到挂载出来到位置编辑配置文件

```bash
vim elasticsearch.yml
```

添加以下几行

```yaml
network.host: 0.0.0.0
discovery.type: single-node

http.cors.enabled: true
http.cors.allow-origin: "*"
```

同时安全组和防火墙记得打开对应端口

> 记得每次修改完配置都要重启

```bash
docker restart elasticsearch
```

> 浏览器访问测试

```bash
http://[IP]:9200
```

看到类似以下的json就成功了

```json
{
	"name": "8c819d377714",
	"cluster_name": "elasticsearch",
	"cluster_uuid": "-AkgwTlbS1SsjvzNtG45nw",
	"version": {
	"number": "7.9.3",
	"build_flavor": "default",
	"build_type": "docker",
	"build_hash": "1c34507e66d7db1211f66f3513706fdf548736aa",
	"build_date": "2020-12-05T01:00:33.671820Z",
	"build_snapshot": false,
	"lucene_version": "8.7.0",
	"minimum_wire_compatibility_version": "6.8.0",
	"minimum_index_compatibility_version": "6.0.0-beta1"
	},
	"tagline": "You Know, for Search"
}
```

> 无法访问

如果开了安全组和防火墙的话还是无法访问到的话，看看在容器启动时是否有如下警告：

```bash
WARNING: IPv4 forwarding is disabled. Networking will not work.
```

按如下步骤操作即可解决，再访问即可

```bash
vim /etc/sysctl.conf

#添加如下代码：
net.ipv4.ip_forward=1

#重启network服务
systemctl restart network

#查看是否修改成功
sysctl net.ipv4.ip_forward

#如果返回为“net.ipv4.ip_forward = 1”则表示成功了
#这时，重启容器即可。
```



### 安装elasticsearch-head (可选)

> 拉取镜像

```bash
docker pull mobz/elasticsearch-head:5
```

> 启动容器

```bash
docker run -it -d --name head -p 9100:9100 mobz/elasticsearch-head:5
```

浏览器打开

```bash
http://[ip]:9100/
```

在上面集群连接处的输入框输入elasticsearch的地址

```bash
http://[IP]:9200
```

之后点击连接，右边的`集群健康值`字样出现绿色背景代表成功连接

### 安装Kibana

> 拉取镜像

```bash
docker pull kibana:7.9.3
```

注意版本和ES的要对应

>  配置文件kibana.yml

为了挂载配置文件，我们先在本机创建一个配置文件，这里以`/home/kibana/config/kibana.yml` 为例

配置文件中写入

```yaml
server.host: '0.0.0.0'
elasticsearch.hosts: ["http://[ip地址]:9200/"]
xpack:
  apm.ui.enabled: false
  graph.enabled: false
  ml.enabled: false
  monitoring.enabled: false
  reporting.enabled: false
  security.enabled: false
  grokdebugger.enabled: false
  searchprofiler.enabled: false
```

> 启动容器

```bash
docker run -d -it \
--name kibana -p 5601:5601 \
-v /home/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml kibana:7.9.3
```

浏览器打开

```bash
http://[ip]:5601/
```

### 安装ik分词器

首先进入es的容器内

```bash
docker exec -it elasticsearch /bin/bash
```

使用bin目录下的elasticsearch-plugin install安装ik分词器插件（注意版本要对应）

```bash
# github官方
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.9.3/elasticsearch-analysis-ik-7.9.3.zip
```

这里可能会很慢，可以用镜像加速

```bash
# 镜像加速
bin/elasticsearch-plugin install https://github.91chifun.workers.dev//https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.9.3/elasticsearch-analysis-ik-7.9.3.zip
```

也可以选择本地下载解压完再上传到服务器，再把它移动到容器内的plugins文件夹里

**然后重启容器**

```bash
docker restart elasticsearch
```

> 在kibana中测试

```bash
GET _analyze
{
  "analyzer": "ik_max_word",
  "text": "各地校车将享最高路权"
}
```

有`ik_smart` 和 `ik_max_word` 两种模式，分别是`最粗粒度的拆分`和`最细粒度的拆分`



## 安全配置——xpack

### 修改配置文件

在`elasticsearch.yml`里新增

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

之后`重启` es

进入容器，再进入bin目录

### 生成密码

在bin目录下执行

```bash
elasticsearch-setup-passwords interactive
```

然后输入多个用户的密码

```bash
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y

Enter password for [elastic]:
Reenter password for [elastic]:
Passwords do not match.
Try again.
Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana]:
Reenter password for [kibana]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

其中`elastic`用户相当与es的root用户，之后使用es和kibana需要这个用户的密码

设置完重启一下es

#### 测试

```bash
curl -GET -u elastic http://[ip]:9200/
```

发现提示输入elastic用户的密码

```bash
Enter host password for user 'elastic':
```

基本的安全就实现了，之后进一步防止暴力破解密码可以再使用`iptables`

### Kibana设置

修改`kibana.yml`

```yaml
elasticsearch.username: "elastic"
elasticsearch.password: "[密码]"
xpack:
  apm.ui.enabled: false
  graph.enabled: false
  ml.enabled: false
  monitoring.enabled: false
  reporting.enabled: false
  security.enabled: true   # 这里要打开
  grokdebugger.enabled: false
  searchprofiler.enabled: false
```

之后进入kibana进入登陆界面

![ElasticSearch%20%E5%AE%89%E5%85%A8%20d5ed4d3e7edc4879bf7a5a55b5c400dc/Untitled.png](https://ccqstark.github.io/p/x_pack/ElasticSearch%20%E5%AE%89%E5%85%A8%20d5ed4d3e7edc4879bf7a5a55b5c400dc/Untitled.png)

用elastic用户和密码登陆即可

### 代码中配置

`Java High Level REST Client`中配置账户和密码

```java
final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
    credentialsProvider.setCredentials(AuthScope.ANY,
            new UsernamePasswordCredentials("elastic", "123456"));  //es账号密码（默认用户名为elastic）
    RestHighLevelClient client = new RestHighLevelClient(
            RestClient.builder(
                    new HttpHost("localhost", 9200, "http"))
                    .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                        public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                            httpClientBuilder.disableAuthCaching();
                            return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                        }
                    }));
```

`SpringBoot`的配置文件

```yaml
spring.elasticsearch.rest.username=elastic
spring.elasticsearch.rest.password=123456
```

### 修改密码

```bash
curl -H "Content-Type:application/json" -XPOST -u elastic 'http://127.0.0.1:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "123456" }'
```



## 索引库结构

>  建立club索引

```json
PUT /club
{
  "mappings": {
    "properties": {
      "clubID":{
        "type": "integer"
      },
      "clubName":{
        "type": "text",
        "analyzer":"ik_max_word"
      },
      "deprecated":{
        "type": "integer"
      },
      "year":{
        "type": "integer"
      }
    }
  }
}
```



> 建立member索引

```json
PUT /member
{
  "mappings": {
    "properties": {
      "memberID":{
        "type": "integer"
      },
      "clubID":{
        "type": "integer"
      },
      "name":{
        "type": "text"
      },
      "duty":{
        "type": "text"
      },
      "sex":{
        "type": "text"
      },
      "college":{
        "type": "text",
        "analyzer":"ik_max_word"
      },
      "classes":{
        "type": "text",
        "analyzer":"ik_max_word"
      },
      "studentNumber":{
        "type": "text"
      },
      "campus":{
        "type": "text"
      },
      "politics":{
        "type": "text"
      },
      "phone":{
        "type": "text"
      },
      "remark":{
        "type": "text",
        "analyzer":"ik_max_word"
      },
      "period":{
        "type": "integer"
      }
    }
  }
}
```

> 清空数据不删除索引结构

```json
POST member/_doc/_delete_by_query?refresh&slices=5&pretty
{
  "query": {
    "match_all": {}
  }
}
```

