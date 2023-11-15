---
layout:     post
title:      SpringCloudKubernetes-02 ConfigMap的引用
subtitle:   SpringCloudKubernetes
date:       2023-09-13
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - SpringCloud
    - ConfigMap
---


此篇文章中，我们将讲述如何从configMap中引入参数配置，如何从挂载文件中引入文件配置。其中文件挂载是应用部署中常见的形式。

- [1、通过 valueRef 引入 ConfigMap 配置信息](#1通过-valueref-引入-configmap-配置信息)
  - [1.1: 初始化项目](#11-初始化项目)
  - [1.2: 定义将外部引入的配置项](#12-定义将外部引入的配置项)
  - [1.3: 构建镜像 \& 发布应用](#13-构建镜像--发布应用)
  - [1.4: 确认配置的引用](#14-确认配置的引用)
- [2、通过 fileAmount 引入 ConfigMap 配置信息](#2通过-fileamount-引入-configmap-配置信息)
  - [2.1: 初始化项目](#21-初始化项目)
  - [2.2: 定义将外部引入的配置项](#22-定义将外部引入的配置项)
  - [2.3: 构建 \& 发布镜像](#23-构建--发布镜像)
  - [2.4: 确认配置的引用](#24-确认配置的引用)

组件版本说明:

- SpringBoot:3.1.0
- SpringCloud:4.0.4
- SpringCloudKubernetes:3.0.4
- JDK17


## 1、通过 valueRef 引入 ConfigMap 配置信息


### 1.1: 初始化项目

引入maven依赖，核心依赖:spring-cloud-kubernetes-fabric8-config

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-kubernetes-fabric8-config</artifactId>
            <version>${springcloud-kubernetes-fabric8.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.22</version>
        </dependency>
    </dependencies>
```

### 1.2: 定义将外部引入的配置项

在 application.yaml 中定义

```xml
spring:
  application:
    name: springboot-cloud-k8s-config-valueref
greeting:
  message: ${GREETING_MESSAGE:nice to meet you}
farewell:
  message: ${FAREWELL_MESSAGE:see you next time}
```

在ConfigMap中定义对应的参数

```xml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-k8s-config-valueref
  labels:
    app: springboot-k8s-config-valueref
data:
  GREETING_MESSAGE: "Say Hello to the World outside"
```

我们定义两个不同的ConfigMap便于更好地演示

```xml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-k8s-config-valueref2
  labels:
    app: springboot-k8s-config-valueref
data:
  farewell.message: "Say Farewell to the World outside"
```

### 1.3: 构建镜像 & 发布应用

通过maven指令编译一个镜像

```xml
 <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.maven.plugin.version}</version>
                <executions>
                    <!--如果想在项目打包时构建镜像添加-->
                    <execution>
                        <id>build-image</id>
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <images>
                        <image>
                            <name>org/${project.artifactId}:${project.version}</name>
                            <build>
                                <dockerFile>${project.basedir}/Dockerfile</dockerFile>
                            </build>
                        </image>
                    </images>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
            <plugin>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
</plugins>
```

可以通过maven指令或IDE的插件

```bash
mvn clean package -Dmaven.test.skip=true
```

执行后，就可以将构建的镜像推送至本地/远程仓库

```bash
REPOSITORY                           TAG                IMAGE ID            CREATED             SIZE
org/springboot-k8s-config-valueref   1.0-SNAPSHOT       8902aa877999        6 seconds ago       564MB
```

通过插件，可以进行应用打包 -> 构建docker镜像 -> 在本地/远程仓库中覆盖更新相同版本的镜像 -> 并清除本地临时文件
接下来，我们可以通过仓库中的镜像，发布一个pod 
这边使用 `configMapRef & configMapKeyRef` 进行引用 :

test.yaml示例文件如下:

```xml
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: springboot-k8s-config-valueref
      labels:
        app: springboot-k8s-config-valueref
    data:
      GREETING_MESSAGE: "Say Hello to the World outside"
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: springboot-k8s-config-valueref2
      labels:
        app: springboot-k8s-config-valueref
    data:
      farewell.message: "Say Farewell to the World outside"
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: springboot-k8s-config-valueref
      name: springboot-k8s-config-valueref
    spec:
      type: NodePort
      selector:
        app: springboot-k8s-config-valueref
      ports:
        - nodePort: 30163
          port: 8080
          protocol: TCP
          targetPort: 8080
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: springboot-k8s-config-valueref
      labels:
        app: springboot-k8s-config-valueref
        group: org.example
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: springboot-k8s-config-valueref
      template:
        metadata:
          labels:
            app: springboot-k8s-config-valueref
        spec:
          volumes:
            - name: autoconfig
          containers:
            - name: springboot-k8s-config-valueref
              image: org/springboot-k8s-config-valueref:1.0-SNAPSHOT
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 8080
              envFrom:
                - configMapRef:
                    name: springboot-k8s-config-valueref
              env:
                - name: FAREWELL_MESSAGE
                  valueFrom:
                    configMapKeyRef:
                      name: springboot-k8s-config-valueref2
                      key: farewell.message
```

执行情况如下：

```bash
$ kubectl apply -f ~/springboot-demo/springboot-config-k8s-valueref/src/main/resources/deploy-valueref.yaml
configmap/springboot-k8s-config-valueref created
configmap/springboot-k8s-config-valueref2 created
service/springboot-k8s-config-valueref created
deployment.apps/springboot-k8s-config-valueref created
$ kubectl get pods
NAME                                              READY   STATUS    RESTARTS   AGE
springboot-k8s-config-valueref-57d464c66c-tg8nw   1/1     Running   0          3s
$ kubectl logs -f springboot-k8s-config-valueref-57d464c66c-tg8nw

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.0)

2023-09-18T12:17:05.206Z  INFO 1 --- [           main] org.example.BootValueRefApplication      : Starting BootValueRefApplication using Java 17.0.8 with PID 1 (/opt/app/springboot-k8s-config-valueref-1.0-SNAPSHOT.jar started by root in /opt/app)
2023-09-18T12:17:05.215Z  INFO 1 --- [           main] org.example.BootValueRefApplication      : The following 1 profile is active: "kubernetes"
2023-09-18T12:17:07.260Z  INFO 1 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=cc47999e-9518-3998-aa7d-05324c2cb413
2023-09-18T12:17:08.015Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2023-09-18T12:17:08.028Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-09-18T12:17:08.029Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.8]
2023-09-18T12:17:08.150Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-09-18T12:17:08.153Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2632 ms
2023-09-18T12:17:09.339Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2023-09-18T12:17:09.412Z  INFO 1 --- [           main] org.example.BootValueRefApplication      : Started BootValueRefApplication in 6.462 seconds (process running for 7.914)
```

经过以上步骤，一个简单的应用已经顺利地跑起来了

### 1.4: 确认配置的引用

这边测试是采用 minikube 在本地搭建 K8S集群。
由于本地环境，需要讲暴露服务来访问，`minikube service springboot-k8s-config-valueref --url`

```bash
minikube service springboot-k8s-config-valueref --url
http://127.0.0.1:51650
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

请求服务API

```bash
$ curl http://127.0.0.1:51650
hello, 'Say Hello to the World outside', goodbye, 'Say Farewell to the World outside'
```

返回在外部ConfigMap中配置的参数，通过configMapRef 和 configMapKeyRef 均可以获取到参数

## 2、通过 fileAmount 引入 ConfigMap 配置信息

以下通过文件挂载的常见形式加载服务变量，通常直接挂载到jar的执行目录下的/config目录，作为springboot加载配置文件的第一优先级，无需指定自动读取

### 2.1: 初始化项目

引入maven依赖，核心依赖:spring-cloud-kubernetes-fabric8-config

main dependency:spring-cloud-kubernetes-fabric8-config

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-kubernetes-fabric8-config</artifactId>
            <version>${springcloud-kubernetes-fabric8.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.22</version>
        </dependency>
    </dependencies>
```

### 2.2: 定义将外部引入的配置项

define the properties in application.yaml, properties in ammounted file not defined.

```yaml
server:
  port: 8080
spring:
  application:
    name: springboot-k8s-config-fileamount
dbuser: ${DB_USERNAME:default}
dbpassword: ${DB_PASSWORD:default}
```

定义 configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-k8s-config-fileamount
  labels:
    app: springboot-k8s-config-fileamount
data:
  application.yaml: |-
    greeting:
      message: "Say Hello to the World outside"
    farewell:
      message: "Say Goodbye to the World outside"
```

定义配置类：

```java
@Data
@Configuration
public class DbConfig {
    @Value("${dbuser}")
    private String dbUsername;
    @Value("${dbpassword}")
    private String dbPassword;
}

@Data
@Configuration
public class MyConfig {

    @Value("${greeting.message:'default greeting message'}")
    private String greetingMessage;
    @Value("${farewell.message:'default farewell message'}")
    private String farewellMessage;
}
```

### 2.3: 构建 & 发布镜像

通过 maven compiler 构建镜像

```xml
 <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.maven.plugin.version}</version>
                <executions>
                    <!--如果想在项目打包时构建镜像添加-->
                    <execution>
                        <id>build-image</id>
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <images>
                        <image>
                            <name>org/${project.artifactId}:${project.version}</name>
                            <build>
                                <dockerFile>${project.basedir}/Dockerfile</dockerFile>
                            </build>
                        </image>
                    </images>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
            <plugin>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
</plugins>
```

通过指令 或 IDE 插件，将镜像打包

```bash
mvn clean package -Dmaven.test.skip=true
```

将镜像推到仓库

```bash
$ docker images
REPOSITORY                                TAG              IMAGE ID       CREATED          SIZE
org/springboot-config-k8s-fileamount      1.0-SNAPSHOT     f93c2e1254d5   42 seconds ago   628MB
```

根据镜像，发布一个Pod
 
使用 `volumes` 引入挂载的文件
使用 `volumeMounts` 指定挂载的文件
针对secret的数据需要在配置前加密，可以使用指定来获取base64加密后的数据： `echo -n root | base64`，然后再填充到文件中

```bash
$ echo -n root | base64
cm9vdA==
```

样例文件：test.yaml:

```yaml
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: springboot-k8s-config-fileamount-secret
      labels:
        app: springboot-k8s-config-fileamount
    data:
      dbuser: cm9vdA==
      dbpassword: cGFzc3dvcmQ=
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: springboot-k8s-config-fileamount
      labels:
        app: springboot-k8s-config-fileamount
    data:
      application.yaml: |-
        greeting:
          message: "Say Hello to the World outside"
        farewell:
          message: "Say Goodbye to the World outside"
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: springboot-k8s-config-fileamount
      name: springboot-k8s-config-fileamount
    spec:
      type: NodePort
      selector:
        app: springboot-k8s-config-fileamount
      ports:
        - nodePort: 30163
          port: 8080
          protocol: TCP
          targetPort: 8080
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: springboot-k8s-config-fileamount
      labels:
        app: springboot-k8s-config-fileamount
        group: org.example
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: springboot-k8s-config-fileamount
      template:
        metadata:
          labels:
            app: springboot-k8s-config-fileamount
        spec:
          volumes:
            - name: config-volume
              configMap:
                name: springboot-k8s-config-fileamount
                items:
                  - key: application.yaml
                    path: application.yaml
          containers:
            - name: springboot-k8s-config-fileamount
              image: org/springboot-config-k8s-fileamount:1.0-SNAPSHOT
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 8080
              env:
                - name: DB_USERNAME
                  valueFrom:
                    secretKeyRef:
                      name: springboot-k8s-config-fileamount-secret
                      key: dbuser
                - name: DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: springboot-k8s-config-fileamount-secret
                      key: dbpassword
              volumeMounts:
                - name: config-volume
                  mountPath: /opt/app/config/application.yaml
                  subPath: application.yaml
```

使用 `kubectl apply -f test.yaml ` 的方式发布Pod

```bash
$ kubectl apply -f ~/springboot-demo/springboot-config-k8s-fileamount/src/main/resources/deploy-fileamount.yaml
secret/springboot-k8s-config-fileamount-secret created
configmap/springboot-k8s-config-fileamount created
service/springboot-k8s-config-fileamount created
deployment.apps/springboot-k8s-config-fileamount created
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
springboot-k8s-config-fileamount-89d6bdfcd-vvkqz   1/1     Running   0          3s
$ kubectl logs springboot-k8s-config-fileamount-89d6bdfcd-vvkqz

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.0)

2023-09-20T10:38:40.477Z  INFO 1 --- [           main] org.example.BootFileamountApplication    : Starting BootFileamountApplication using Java 17.0.8 with PID 1 (/opt/app/springboot-config-k8s-fileamount-1.0-SNAPSHOT.jar started by root in /opt/app)
2023-09-20T10:38:40.480Z  INFO 1 --- [           main] org.example.BootFileamountApplication    : The following 1 profile is active: "kubernetes"
2023-09-20T10:38:42.402Z  INFO 1 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=49502b73-ffac-39f1-a249-dec4d3facce7
2023-09-20T10:38:43.379Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2023-09-20T10:38:43.402Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-09-20T10:38:43.403Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.8]
2023-09-20T10:38:43.517Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-09-20T10:38:43.520Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2845 ms
2023-09-20T10:38:44.714Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2023-09-20T10:38:44.792Z  INFO 1 --- [           main] org.example.BootFileamountApplication    : Started BootFileamountApplication in 6.37 seconds (process running for 8.022)
```

应用启动，准备测试

### 2.4: 确认配置的引用

暴露端口 `minikube service springboot-k8s-config-valueref --url`

```bash
minikube service springboot-k8s-config-fileamount --url
http://127.0.0.1:55369
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

请求接口，获取注入的参数值

```bash
$ curl http://127.0.0.1:55369
hello, 'Say Hello to the World outside', goodbye, 'Say Goodbye to the World outside'. myname: 'root', mypass: 'password' 
```

参数值获取成功注入的参数