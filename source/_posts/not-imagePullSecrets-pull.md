---
title: 配置免imagePullSecrets拉取私有仓库镜像
categories:
  - [kubernetes]
tags:
    - kubernetes
date: 2023-02-04
---
> 在kubernetes中的pod运行的镜像有时候会存储在企业搭建的私有镜像仓库，拉取这类镜像的时候一般需要登录授权后才能拉取，这时候有两种方式实现：

> 方法一：在deployment或pod中配置imagePullSecrets声明拉取镜像时需要使用的账号密码secret，但是需要每个deployment和pod都要配置该字段，比较繁琐

> 方法二：给namespace下的default serviceaccount添加拉取镜像的账号密码secret，这样整个namespace下的deployment和pod拉取镜像是使用该serviceaccount下的secret去拉取镜像

以下就是方法二的设置方式：

# 步骤1：生成拉取docker镜像的用户名密码的secret

### 生成用户名密码base64字符串

```shell
#假设用户名是:"admin"，密码是："123456",构建"admin:123456"这样的字符串，然后base64编码
echo "admin:123456"|base64
YWRtaW46MTIzNDU2Cg==
```

### 创建配置文件docker-config.json，格式如下：

```text
{
  "auths":
    {
        "registry.cn-shanghai.aliyuncs.com":
        {   
            "auth":"YWRtaW46MTIzNDU2Cg=="
        },
        "registry-vpc.cn-shanghai.aliyuncs.com":
        {
            "auth":"YWRtaW46MTIzNDU2Cg=="
        }
    }
}
```

> 以上文件配置了registry.cn-shanghai.aliyuncs.com和registry-vpc.cn-shanghai.aliyuncs.com这两个需要登录的docker仓库的登录用户和密码

> auth是用于登录仓库的"username:password"进行base64编码字符串

### 编码docker-config.json文件：

```shell
base64 docker-config.json
ewogICJhdXRocyI6CiAgICB7CiAgICAgICAgInJlZ2lzdHJ5LmNuLXNoYW5naGFpLmFsaXl1bmNz
LmNvbSI6CiAgICAgICAgeyAgIAogICAgICAgICAgICAiYXV0aCI6IllXUnRhVzQ2TVRJek5EVTJD
Zz09IgogICAgICAgIH0sCiAgICAgICAgInJlZ2lzdHJ5LXZwYy5jbi1zaGFuZ2hhaS5hbGl5dW5j
cy5jb20iOgogICAgICAgIHsKICAgICAgICAgICAgImF1dGgiOiJZV1J0YVc0Nk1USXpORFUyQ2c9
PSIKICAgICAgICB9CiAgICB9Cn0K
```

将base64编码的字符用于后续生成kubernetes的资源secret

### 使用上一步得到的docker-config.json的base64编码字符串创建docker-secret.yaml：

```yaml
apiVersion: v1
data:
  .dockerconfigjson: ewogICJhdXRocyI6CiAgICB7CiAgICAgICAgInJlZ2lzdHJ5LmNuLXNoYW5naGFpLmFsaXl1bmNz
LmNvbSI6CiAgICAgICAgeyAgIAogICAgICAgICAgICAiYXV0aCI6IllXUnRhVzQ2TVRJek5EVTJD
Zz09IgogICAgICAgIH0sCiAgICAgICAgInJlZ2lzdHJ5LXZwYy5jbi1zaGFuZ2hhaS5hbGl5dW5j
cy5jb20iOgogICAgICAgIHsKICAgICAgICAgICAgImF1dGgiOiJZV1J0YVc0Nk1USXpORFUyQ2c9
PSIKICAgICAgICB9CiAgICB9Cn0K
kind: Secret
metadata:
  name: docker-secret
type: kubernetes.io/dockerconfigjson
```

！！！注意：data的key名称必须是".dockerconfigjson"

### 使用kubectl创建secret:

```shell
kubectl -n dev-appkyhk -f docker-secret.yaml
```

> 该命令在kubernetes的devappkyhk命名空间下创建了 docker-secret的secret。该secret是d拉取私有进行仓库的的用户名密码配置。

# 步骤2：绑定docker-secret到指定命名空间下的default的ServiceAccount

```shell
kubectl patch serviceaccount default -n dev-appkyhk -p '{"imagePullSecrets": [{"name": "docker-secret"}]}'
```

> 该命令在dev-appkyhk这个命名空间下的default这个serviceacount的添加了imagePullSecrets: docker-secret属性。设置后该命名空间下所有deployment和pod在拉取对应镜像的时候默认都会使用docker-secret这个secret中存放的用户名密码进行登录授权。

**完**