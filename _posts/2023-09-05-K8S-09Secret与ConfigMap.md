---
layout:     post
title:      K8S学习笔记|09-Secret&ConfigMap
subtitle:   Kubernetes
date:       2023-09-05
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - K8S
    - Secret
    - ConfigMap
---

- [Secret](#secret)
  - [通过Volume的方式，将Secret挂载到容器](#通过volume的方式将secret挂载到容器)
  - [通过环境变量的方式将Secret挂载到容器](#通过环境变量的方式将secret挂载到容器)
- [ConfigMap](#configmap)
  - [通过volume方式被应用引用](#通过volume方式被应用引用)
  - [通过环境变量的方式被应用引用](#通过环境变量的方式被应用引用)
  - [通过 YAML 指定KV配置](#通过-yaml-指定kv配置)

## Secret

通过 `--from-literal` 显示指明 `key=value`的值
一条声明对应一个kv对

```bash
kubectl create secret generic mysecret --from-literal=user=root --from-literal=pass=password
```

通过 `--from-file` 指明配置文件
一个文件对应一个kv对，文件名是key，内容是value

```bash
kubectl create secret generic mysecret --from-file=/user
--from-file=/pass
```

通过 `--from-env-file` 指定
一个文件包含多个kv对

```bash
kubectl create secret generic mysecret --from-env-file=env.txt
```

文件内就是:

```text
user=root
pass=password
```

通过YAML文件指定Secret配置，其中可以定义KV对，定义KV列表，定义文件内容

```yml
apiVersion:v1
kind: Secret
metadata:
  name: mysecret
data:
  user: base64(root) -- 需要加密，保证安全
  pass: base64(password) -- 需要加密，保证安全
```

最终通过`kubectl apply` 将文件执行，创建Secret

```bash
kubectl get secret mysecret  -- 查看是否创建成功
kubectl describe secret mysecret  -- 查看创建的key
kubectl edit secret mysecret -- 查看内容/修改内容
```

明确的数据，需要将base64的数据decode，即可查看到明文

以上已经能够将日常应用所需的一些重要账密密文存储起来，**那么应用如何引用呢？**

### 通过Volume的方式，将Secret挂载到容器

可以通过将密钥挂载到应用的目录，应用读取的方式进行使用，内部路径上，默认文件名为key,内容为value明文。key名字可以在YAML文件中显示指定。
外部secret更新，容器内部可以同步更新

### 通过环境变量的方式将Secret挂载到容器

```yml
env:
  name: MYSQL_USERNAME
  valueFrom:
    secretKeyRef:
      name: mysecret
      key: user
  name: MYSQL_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysecret
      key:pass
```

Pod内部，通过`$MYSQL_USERNAME`，`$MYSQL_PASSWORD` 直接访问明文
环境变量方式，无法动态更新Secret

通常敏感数据需要Secret进行加密，此外一个Pod之外和环境相关的普通配置文件，则可以使用ConfigMap的方式在外部进行配置

## ConfigMap

配置的方式与Secret类似，存储为明文
通过 `--from-literal` 进行kv指定
通过 `--from-file` 进行单key文件指定
通过 `--from-env-file` 进行多kv文件指定
通过YAML文件，配置为明文，格式同Secret

### 通过volume方式被应用引用

### 通过环境变量的方式被应用引用

格式同Secret，通过configMapKeyRef进行引用

通常configmap中信息较多，常见的是通过volume的方式进行挂载，支持动态更新

通过 `--from-file` 指定kv配置文件

```bash
kubectl create configmap myconfigmap --from-file=./logging.conf
```

```yml
level:INFO
formatter: precise
```

### 通过 YAML 指定KV配置

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  logging.conf: |
    formatter: precise
    level:INFO
```

```bash
kubectl apply -f myconfigmap.yaml
```

The End:

- 敏感信息用Secret
- 普通配置信息用ConfigMap
- 通过4种方式可以指定配置信息，通常使用volume方式挂载配置，支持动态更新，使用环境变量的方式，免去引用读取文件，但不支持动态更新
