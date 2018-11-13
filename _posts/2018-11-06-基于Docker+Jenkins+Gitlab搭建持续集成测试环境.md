---
layout: post
title: 基于Docker+Jenkins+Gitlab搭建持续集成测试环境
categories: 随编
tags: 容器

---

　　七牛云的测试域名到期了，所有图片的图床都崩了，早知如此不该图方便，自己做静态资源算了。进入今天的正题，到新公司一个月，项目开发前后端分离，差不多拉通了开发到测试的流程，在此记录一下。

　　随着DevOps理念和敏捷理念的发展，我们希望通过自动化技术，加快项目的迭代。尤其是当使用微服务方案后，面临在大量的项目构建和部署工作，借助于jenkins的持续集成，可以快速把应用打包成docker镜像，实现自动部署。

![](https://ixjx.github.io/blog/static/img/jenkins.png)

如图演示了以下的场景：

- 开发者向自己的gitlab提交了代码

- jenkins通过定时任务检测到了代码有变成，执行自动化构建过程

- jenkins在自动化构建脚本中调用docker命令将构建好的镜像push到私有镜像中心harbor

- 同时，jenkins也可以直接执行remote shell启动构建好的容器

- 构建失败或者成功，可以及时将结果推送给相关人员，比如测试人员，安排测试

- 服务端可以手动通过docker命令，从镜像仓库中心拉取镜像，进行手动部署


> 环境如下:

| 192.168.110.202 | harbor |
| 192.168.110.203 | gitlab jenkins |

　　除了jenkins均采用docker部署。

## 1. 搭建harbor

~~docker run -d -p 5000:5000 -v /opt/docker-registry:/var/lib/registry registry~~ 一开始用的registry，连个UI都没有，使用不便，弃了


[harbor官方安装文档](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)

采用offline安装包,在执行./prepare的时候抛出如下异常：

```bash
root@ubuntu:~/harbor# ./prepare 
Fail to generate key file: ./common/config/ui/private_key.pem, cert file: ./common/config/registry/root.crt
```

需要修改prepare文件，将第498行：

`empty_subj = "/C=/ST=/L=/O=/CN=/"`

修改如下：

`empty_subj = "/C=US/ST=California/L=Palo Alto/O=VMware, Inc./OU=Harbor/CN=notarysigner"`

配置daemon.json，去掉docker(每个docker client都需要配置)默认的https的访问`vim /etc/docker/daemon.json`

里面的内容是一个json对象,加上一项insecure-registries，地址自己更改：

```
{
    "insecure-registries":["192.168.1.78"]
}
```

然后重启docker,执行

`systemctl daemon-reload`

`systemctl restart docker`

## 2. 搭建gitlab

```bash
docker run --detach \
--hostname localhost \
--publish 443:443 --publish 80:80 \
--name gitlab \
--restart always \
--volume /opt/gitlab/config:/etc/gitlab \
--volume /opt/gitlab/logs:/var/log/gitlab \
--volume /opt/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

　　之后push demo代码。

## 3. jenkins安装

　　这一块比较复杂，不讲了

## 4. 打包前端demo，jenkins构建shell

　　前端用npm打包，后端用的maven。

```bash
#!/bin/sh
source /etc/profile
cnpm cache verify
cnpm install
cnpm run build
tar -zcf web.tar.gz dist
rm -rf dist
echo "******打包完成******"


#!/bin/sh
REGISTRY="192.168.110.202"
REPO="library"
HARBOR_USER="admin"
HARBOR_PASSWD="Harbor12345"
API_NAME="demo"
API_TAG="1.0"
IMAGE_NAME="${REGISTRY}/${REPO}/${API_NAME}:${API_TAG}"
CONTAINER_NAME="${API_NAME}-v${API_TAG}"

docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWD} ${REGISTRY}

tar -zxf web.tar.gz
docker build -t $IMAGE_NAME .
docker push $IMAGE_NAME
echo "******镜像推送完成******"


cid=$(docker ps -a | grep "$CONTAINER_NAME" | awk '{print $1}')
if [ -n "$cid" ];then
	docker rm -f $cid
fi
echo "******删除旧容器完成******"

iid=$(docker images | grep "none" | awk '{print $3}')
if [ -n "$iid" ];then
	docker rmi $iid
fi
echo "******删除旧镜像完成******"

docker run -d -p 2015:2015 --name $CONTAINER_NAME $IMAGE_NAME
echo "******本地部署新容器完成******"


#!/bin/sh
REGISTRY="192.168.110.202"
REPO="library"
HARBOR_USER="admin"
HARBOR_PASSWD="Harbor12345"
API_NAME="demo"
API_TAG="1.0"
IMAGE_NAME="${REGISTRY}/${REPO}/${API_NAME}:${API_TAG}"
OLD_IMAGE_NAME="${REGISTRY}/${REPO}/${API_NAME}"
CONTAINER_NAME="${API_NAME}-v${API_TAG}"

docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWD} ${REGISTRY}


cid=$(docker ps -a | grep "$CONTAINER_NAME" | awk '{print $1}')
if [ -n "$cid" ];then
	docker rm -f $cid
fi
echo "******删除旧容器完成******"

iid=$(docker images | grep "$OLD_IMAGE_NAME" | awk '{print $3}')
if [ -n "$iid" ];then
	docker rmi $iid
fi
echo "******删除旧镜像完成******"

docker pull $IMAGE_NAME
echo "******拉取新镜像完成******"

docker run -d -p 2015:2015 --name $CONTAINER_NAME $IMAGE_NAME
echo "******远程部署新容器完成******"

```

## 5. 问题

1. gitlab需通过root账号登录，允许本地webhook。这是因为gitlab和jenkins在同一个节点部署，实际分开部署不会有这个问题

```bash
gitlab-rails console production   #gitlab居然是用ruby写的
u = User.where(email: 'admin@example.com').first
u.password='new_password'
u.save!
```

gitlab配置Admin area：

Setting->Network->Outbound requests

勾选Allow requests to the local network from hooks and services

jenkins配置：

Jenkins-->Jenkins Manages-->Configure System，找到GitLab配置，去掉勾选

系统管理-->全局安全配置，勾选匿名用户具有可读权限和去掉CSRF防止跨站点请求伪造