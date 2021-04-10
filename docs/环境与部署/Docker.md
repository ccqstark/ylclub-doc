## Docker安装

> CentOS下安装 

### 添加下载源

```bash
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

### 下载安装

```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

### 启动docker

```bash
sudo systemctl start docker
```

### 验证是否成功

```bash
sudo docker run hello-world
```

> 出现以下字样代表成功

```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```



## 各容器部署

### Jenkins

#### 安装启动

```bash
docker pull jenkins/jenkins

docker run -u root -itd --name jenkins \
-p 6001:8080 \
-v $(which docker):/usr/bin/docker \
-v /var/run/docker.sock:/var/run/docker.sock -e TZ="Asia/Shanghai" \
-v /etc/localtime:/etc/localtime:ro \
-v /volume1/docker/jenkins:/var/jenkins_home \
jenkins/jenkins
```

- `-p 6001:8080`Jenkins默认网页访问端口为8080，将端口映射到外部主机6001端口
- `-v $(which docker):/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock`使Jenkins内部可以使用docker命令
- `-e TZ="Asia/Shanghai" -v /etc/localtime:/etc/localtime:ro`配置Jenkins容器的时区
- `-v /volume1/docker/jenkins:/var/jenkins_home` 将Jenkins的配置映射到外部主机卷，容器删除仍可保留配置

#### 测试Jenkins容器内部

```bash
# 进入Jenkins的容器内部
docker exec -it jenkins bash

# 判断docker命令是否正常执行
docker info
```

用`http://主机IP:6001` 就可以访问Jenkins的网页端了





### MySQL





### Redis







## 镜像构建





