# Docker入门

## Java应用容器化最佳实践

本文以多模块Gradle项目为例，进行一次容器化实战最贱实践，

容器化的期望：

1. 构建产物足够小
2. 构建速度足够快

本文用到的软件版本如下:

| 软件     | 版本    | 备注            |
| ------ | ----- | ------------- |
| Java   | 1.8   |               |
| Docker | 20    | 17+版本支持了多阶段构建 |
| Gradle | 7.4.2 |               |

应用容器化方案

| 方案                            | 配置形式           | 优势                                            | 劣势                                 |
| ----------------------------- | -------------- | --------------------------------------------- | ---------------------------------- |
| Docker multi stage            | Dockerfile     | <p>1.构建自由度高<br>2.</p>                         | <p>1. Dockerfile编写繁琐<br>2.<br></p> |
| Buildpacks                    | Proc           | 1.针对于小白上手难度低                                  | <p>1. 构建产物体积大<br>2.<br></p>        |
| Jib                           | gradle/maven配置 | <p>1.配置简单，无需编写其他的Dockerfile<br>2.<br><br></p> |                                    |
| BuildKit                      | Dockerfile     | 1.并发构建快                                       |                                    |
| Docker multi stage + BuildKit |                |                                               |                                    |

#### 如何构建一个优质的Docker镜像？

1. 从适当的基础映像开始。例如，如果您需要 JDK，请考虑将您的镜像基于官方`openjdk`镜像，而不是从通用`ubuntu`镜像开始并`openjdk`作为 Dockerfile 的一部分进行安装。
2. [使用多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)。例如，您可以使用该`maven`映像来构建您的 Java 应用程序，然后重置为该`tomcat`映像并将 Java 工件复制到正确的位置以部署您的应用程序，所有这些都在同一个 Dockerfile 中。这意味着您的最终映像不包含构建中引入的所有库和依赖项，而仅包含运行它们所需的工件和环境。
3. 如果您有多个具有很多共同点的图像，请考虑使用共享组件创建您自己的 [基础图像](https://docs.docker.com/develop/develop-images/baseimages/)，并以此为基础创建您的独特图像。Docker 只需要加载一次公共层，并且将它们缓存起来。这意味着您的衍生镜像更有效地使用 Docker 主机上的内存并更快地加载。
4. 要保持生产映像精简但允许调试，请考虑使用生产映像作为调试映像的基础映像。可以在生产映像之上添加额外的测试或调试工具。
5. 在构建映像时，始终使用有用的标签来标记它们，这些标签会编码版本信息、预期目标（`prod`或`test`，例如）、稳定性或在不同环境中部署应用程序时有用的其他信息。不要依赖自动创建的`latest`标签。

Docker官网最佳实践建议：https://docs.docker.com/develop/dev-best-practices/

本文推荐的基础Linux镜像是alpine，因为alpine的体积足够小，体积甚至缩到了惊人的5-6MB之间，使用alpine作为基础镜像来说体积方面具有很大优势。

alpine镜像github地址：https://github.com/alpinelinux/docker-alpine

#### 准备工作

1. 准备一台Linux机器作为构建机器；
2. 安装Docker 20.10.11版本
3. 安装Buildpacks
4. 安装BuildKit
5. 准备一个多模块的Gradle项目

Demo项目结构

```shell
├── Dockerfile                    容器化Docker文件                  
├── Dockerfile.cache							 Gradle缓存容器化Docker文件                  
├── backend-gateway   						 子模块                  
│   └── src   						 					源码目录
├── build.gradle.kts   						 全局项目依赖生命文件
├── buildimage.sh 								 容器化脚本
├── gradlew 												gradle命令脚本
└── settings.gradle.kts						 gradle项目设置文件
```

Demo项目地址：TODO 贴上github地址

#### 前提说明

为了更客观的测试以上方案的优劣，增加以下前提

1. 提前下载构建过程中可能用到的基础镜像
2. 使用依赖缓存镜像来加速Java应用的构建

#### Docker multi stage构建

TODO 画图多阶段构建

依赖缓存基础镜像Dockerfile.cache（文件名）

```dockerfile
FROM gradle:jdk8 as builder
WORKDIR /app
COPY ["build.gradle.kts","settings.gradle.kts","gradle.properties","./"]
RUN gradle clean build -x bootJar -i --stacktrace --no-daemon && cp -r /root/.gradle/ /root/gradle_cache

FROM gradle:jdk8 as builder
COPY --from=builder /root/gradle_cache /root/.gradle
```

构建指令 TODO. 使用github地址代替

```shell
docker build -f ./Dockerfile.cache -t gradle-cache:latest .
```

构建的依赖缓存镜像如下

```shell
# docker images --filter=reference=gradle-cache:latest
REPOSITORY     TAG       IMAGE ID       CREATED       SIZE
gradle-cache   latest    e6d630b62a0c   5 hours ago   997MB
```

Java应用容器化Dockerfile（文件名）

```dockerfile
ARG CACHE_IMAGE

FROM ${CACHE_IMAGE} as jarBuilder
WORKDIR  /app
COPY  ./ /app
RUN gradle :backend-gateway:bootJar -i --stacktrace --no-daemon

FROM azul/zulu-openjdk-alpine:11 as packager
ENV JAVA_MINIMAL=/opt/jre
WORKDIR /app
COPY --from=jarBuilder /app/build/*.jar  .
RUN jlink \
    --verbose \
    --add-modules \
        java.base,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument \
    --compress 2 \
    --strip-debug \
    --no-header-files \
    --no-man-pages \
    --output "$JAVA_MINIMAL"

FROM alpine
WORKDIR /app
ENV JAVA_MINIMAL=/opt/jre
ENV PATH="$PATH:$JAVA_MINIMAL/bin"
COPY --from=packager "$JAVA_MINIMAL" "$JAVA_MINIMAL"
COPY --from=jarBuilder /app/*.jar  .
CMD ["java", "-jar", "backend-gateway-1.0-boot.jar"]
```

构建指令 TODO. 使用github地址代替 docker build GitHub.....详细参考https://docs.docker.com/engine/reference/commandline/build/

```shell
# docker images --filter=reference="backend-gateway:1.0"
REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
backend-gateway   1.0       4986f3640885   35 seconds ago   103MB
```

TODO

1 多阶段构建会产生很多中间镜像

2 是否应该保存中间镜像？不保留怎么清除中间镜像

#### Buildpacks

buildpacks文档：https://buildpacks.io/docs/

heroku构建gradle项目指南文档：https://devcenter.heroku.com/articles/deploying-gradle-apps-on-heroku

查看buildpacks推荐的builder

```shell
# pack builder suggest
Suggested builders:
        Google:                gcr.io/buildpacks/builder:v1      Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python                                                      
        Heroku:                heroku/buildpacks:20              Base builder for Heroku-20 stack, based on ubuntu:20.04 base image                                                                
        Paketo Buildpacks:     paketobuildpacks/builder:base     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Python, Ruby, NGINX and Procfile                        
        Paketo Buildpacks:     paketobuildpacks/builder:full     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Python, PHP, Ruby, Apache HTTPD, NGINX and Procfile     
        Paketo Buildpacks:     paketobuildpacks/builder:tiny     Tiny base image (bionic build image, distroless-like run image) with buildpacks for Java, Java Native Image and Go                

Tip: Learn more about a specific builder with:
```

Procfile文件内容

```shell
java -Dserver.port=8080 -jar build/backend-gateway-1.0-boot.jar
```

构建指令

```shell
pack build --path=./ --builder heroku/buildpacks:20 backend-gateway:latest
```

#### Jib构建

docker login https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#what-should-i-do-when-the-registry-responds-with-unauthorized

```kotlin
plugins {
    id("com.google.cloud.tools.jib") version "3.2.1"
}
```

```shell
gradle :backend-gateway:jib --image=ccr.ccs.tencentyun.com/tcb-2446011668-egvs/backend-gateway
```

BuildKit构建：





