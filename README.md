# Docker入门



## Java应用容器化最佳实践

本文以多模块Gradle项目为例，进行一次容器化最佳实践，

容器化的期望：

1. 构建产物足够小
2. 构建速度足够快

本文用到的软件版本如下:

| 软件         | 版本                            | 备注            |
| ---------- | ----------------------------- | ------------- |
| Java       | 1.8                           |               |
| Docker     | 20                            | 17+版本支持了多阶段构建 |
| Gradle     | 7.4.2                         |               |
| Buildpacks | 0.25.0+git-b9be13a.build-3254 |               |

应用容器化方案

| 方案                 | 配置形式           | 优势                                                                           | 劣势                                 |
| ------------------ | -------------- | ---------------------------------------------------------------------------- | ---------------------------------- |
| Docker multi stage | Dockerfile     | <p>1.构建自由度高<br>2.</p>                                                        | <p>1. Dockerfile编写繁琐<br>2.<br></p> |
| Buildpacks         | Procfile       | <p>1.针对于小白上手难度低<br>2.自动检测使用框架技术和jdk</p>                                      | <p>1. 构建产物体积大<br>2.<br></p>        |
| Jib                | gradle/maven配置 | <p>1.配置简单，无需编写其他的Dockerfile<br>2.jib在构建过程中会自定把镜像push到ccr仓库<br>3.<br><br></p> |                                    |
| BuildKit           | Dockerfile     | <p>1.并发构建快<br>2.通过挂载本地文件系统可以缓存依赖<br></p>                                     |                                    |
|                    |                |                                                                              |                                    |

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
5. 安装Gradle
6. 准备一个Gradle多模块项目，本文示例项目github地址：https://github.com/Maple-mxf/microservices-backend

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

#### 前提说明

为了更客观的测试以上方案的优劣，增加以下前提

1. 提前下载构建过程中可能用到的基础镜像
2. 使用依赖缓存镜像来加速Java应用的构建
3. 提前在构建机器上登陆dockerhub，来避免因为高频率拉镜像被限流

#### Docker multi stage构建

依赖缓存基础镜像Dockerfile.cache（文件名）

```dockerfile
FROM gradle:jdk8 as builder
WORKDIR /app
COPY ["build.gradle.kts","settings.gradle.kts","gradle.properties","./"]
RUN gradle clean build -x bootJar -i --stacktrace --no-daemon && cp -r /root/.gradle/ /root/gradle_cache

FROM gradle:jdk8 as builder
COPY --from=builder /root/gradle_cache /root/.gradle
```

构建指令

```shell
  docker build \
  --file Dockerfile.cache \
  --tag  gradle-cache:latest  https://github.com/Maple-mxf/microservices-backend.git
```

note:构建上下文设定为github仓库，详情参考[docker build指南](https://docs.docker.com/engine/reference/commandline/build/)

构建的依赖缓存镜像如下

```shell
# docker images --filter=reference=gradle-cache:latest
REPOSITORY     TAG       IMAGE ID       CREATED       SIZE
gradle-cache   latest    e6d630b62a0c   5 hours ago   997MB
```

Java应用容器化Dockerfile（文件名）

如下Dockerfile为最终构建gradle模块的Dockerfile

1. CACHE\_IMAGE动态表示缓存的gradle依赖的镜像
2. MODULE表示要构建的gradle模块名称
3. 在执行构建命令时通过build-args选项参数注入对应的值

```dockerfile
ARG CACHE_IMAGE

FROM ${CACHE_IMAGE} as jarBuilder
ARG MODULE
WORKDIR  /app
COPY  ./ /app
RUN gradle :${MODULE}:bootJar -i --stacktrace --no-daemon && ls -a

FROM alpine:3.13
ARG MODULE
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tencent.com/g' /etc/apk/repositories \
    && apk add --update --no-cache openjdk8-jre-base \
    && rm -f /var/cache/apk/*
WORKDIR /app
COPY --from=jarBuilder /app/build/${MODULE}-*-boot.jar  ./app.jar
CMD ["sleep", "1h"]
```

_Note:Dockerfile的Arg标记有生效范围，如果只在FROM指令声明，则只在FROM中生效_

构建指令

```shell
docker build \
--no-cache  \
--build-arg MODULE="backend-gateway" \
--build-arg CACHE_IMAGE="gradle-cache:latest" \
--tag "backend-gateway":"3.0"  https://github.com/Maple-mxf/microservices-backend.git \
&& docker image prune -f 
```

查看docker构建好的产物

```shell
# docker images --filter=reference="backend-gateway:3.0"
REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
backend-gateway   3.0       36eb9bc545b4   4 minutes ago   124MB
```

_Notes:由于docker multi stage构建会产生很多的中间镜像，所以执行了 docker image prune -f 清除中间的多余镜像_

#### Buildpacks

buildpacks文档：https://buildpacks.io/docs/

heroku构建gradle项目指南文档：https://devcenter.heroku.com/articles/deploying-gradle-apps-on-heroku

buildpacks属于一款重量级自动构建工具，无需编写Dockerfile

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

构建输出

```shell
20: Pulling from heroku/buildpacks
Digest: sha256:e7d4a2fce9218495c599bf2d5f790632d81fae1a4872f92fcdfababa6a33b415
Status: Image is up to date for heroku/buildpacks:20
20: Pulling from heroku/pack
Digest: sha256:4559f682a6a7c57a4f1252a85c08d052e9e748126101938f28d37bd5ff63cf3b
Status: Image is up to date for heroku/pack:20
===> ANALYZING
Previous image with name "backend-gateway:20220503175029" not found
===> DETECTING
heroku/gradle   0.0.35
heroku/procfile 0.6.2
===> RESTORING
===> BUILDING
-----> Spring Boot detected
-----> Installing JDK 11... done
-----> Building Gradle app...
-----> executing ./gradlew build -x check
       Downloading https://services.gradle.org/distributions/gradle-7.4.2-bin.zip
       ...........10%...........20%...........30%...........40%...........50%...........60%...........70%...........80%...........90%...........100%
       To honour the JVM settings for this build a single-use Daemon process will be forked. See https://docs.gradle.org/7.4.2/userguide/gradle_daemon.html#sec:disabling_the_daemon.
       Daemon will be stopped at the end of the build 
       > Task :compileJava NO-SOURCE
       > Task :processResources NO-SOURCE
       > Task :classes UP-TO-DATE
       > Task :bootJar SKIPPED
       > Task :jar SKIPPED
       > Task :assemble SKIPPED
       > Task :build SKIPPED
       > Task :backend-gateway:compileJava
       > Task :backend-gateway:processResources
       > Task :backend-gateway:classes
       > Task :backend-gateway:bootJar
       > Task :backend-gateway:jar SKIPPED
       > Task :backend-gateway:assemble
       > Task :backend-gateway:build
       
       BUILD SUCCESSFUL in 1m 36s
       3 actionable tasks: 3 executed
[INFO] Discovering process types
[INFO] Procfile declares types -> web
===> EXPORTING
Adding layer 'heroku/gradle:profile'
Adding 1/1 app layer(s)
Adding layer 'launcher'
Adding layer 'config'
Adding layer 'process-types'
Adding label 'io.buildpacks.lifecycle.metadata'
Adding label 'io.buildpacks.build.metadata'
Adding label 'io.buildpacks.project.metadata'
Setting default process type 'web'
Saving backend-gateway:20220503175029...
*** Images (76d79a8ab094):
      backend-gateway:20220503175029
Adding cache layer 'heroku/gradle:shim'
Successfully built image backend-gateway:20220503175029
Build success, buildType=buildpacks,costTime=2m11s
```

查看构建完成后的产物

```shell
# docker images --filter=reference="backend-gateway:20220503175029"
REPOSITORY        TAG              IMAGE ID       CREATED        SIZE
backend-gateway   20220503175029   76d79a8ab094   42 years ago   802MB
```

指定Java版本，参考文档[buildpacks文档](https://devcenter.heroku.com/articles/java-support#specifying-a-java-version)

项目根目录下新增system.properties

```properties
java.runtime.version=11
```

_Note:buildpacks构建的缺点也很明显，会默认构建所有的gradle项目模块_

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

构建指令

```shell
gradle :gateway-backend:jib --image=ccr.tencent.com/2446011668/backend-gateway:1.0
```

查看产物大小

jib构建日志

```shell
Containerizing application to ccr.ccs.tencentyun.com/tcb-2446011668-egvs/backend-gateway:20220503180517...
Base image 'openjdk:alpine' does not use a specific image digest - build may not be reproducible
Using credentials from Docker config (/root/.docker/config.json) for ccr.ccs.tencentyun.com/tcb-2446011668-egvs/backend-gateway:20220503180517
The base image requires auth. Trying again for openjdk:alpine...
Using credentials from Docker config (/root/.docker/config.json) for openjdk:alpine
Using base image with digest: sha256:1fd5a77d82536c88486e526da26ae79b6cd8a14006eb3da3a25eb8d2d682ccd6

Container entrypoint set to [java, -Xms512m, -Xdebug, -cp, /app/resources:/app/classes:/app/libs/*, backend.router.BackendRouterApp]

Built and pushed image as ccr.ccs.tencentyun.com/tcb-2446011668-egvs/backend-gateway:20220503180517
Executing tasks:
[==============================] 100.0% complete


BUILD SUCCESSFUL in 16s
3 actionable tasks: 3 executed
Build success, buildType=jib,costTime=17s
```

jib构建速度快，并且jib会自动的把镜像推到仓库，只需要将--image参数换成对应的container registry

jib自定义配置. 参考[jib gradle配置文档](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#quickstart)

```
   jib {
        from {
            image = "openjdk:alpine"
        }
        container {
            mainClass = "backend.router.BackendRouterApp"
            ports = mutableListOf("8080")
            jvmFlags = mutableListOf("-Xms512m", "-Xdebug")
            format = com.google.cloud.tools.jib.api.buildplan.ImageFormat.OCI
        }
    }
```

jib插件扩展. 参考[jib extension开发文档](https://github.com/GoogleContainerTools/jib-extensions)

#### BuildKit构建

buildkit是下一代的构建工具

buildkit项目github地址：https://github.com/moby/buildkit

Docker开启buildkit支持，在/etc/docker/daemon.json增加以下配置

```json
{"features": { "buildkit": true }}
```

由于Buildkit是实验特性，并且需要在Dockerfile首行中增加以下注释

```dockerfile
# syntax = docker/dockerfile:experimental
```

Dockerfile

```dockerfile
# syntax = docker/dockerfile:experimental
FROM gradle:jdk8 as jarBuilder
WORKDIR /app
COPY . ./
RUN --mount=type=cache,target=~/.gradle/,id=gradle_repo_cache,sharing=locked
ARG MODULE
RUN gradle :${MODULE}:bootJar -i --stacktrace --no-daemon

FROM alpine:3.13
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tencent.com/g' /etc/apk/repositories \
    && apk add --update --no-cache openjdk8-jre-base \
    && rm -f /var/cache/apk/*
WORKDIR /app
ARG MODULE
COPY --from=jarBuilder /app/build/${MODULE}-*-boot.jar  ./app.jar
CMD ["sleep", "1h"]
```

执行构建命令

```shell
docker build \
--no-cache  \
--build-arg MODULE=backend-gateway \
--file Dockerfile.buildkit \
--tag backend-gateway:buiuldkit  https://github.com/Maple-mxf/microservices-backend
```

构建方案对比

| 构建方式               | 耗时      | 产物大小  | 制品类型         |
| ------------------ | ------- | ----- | ------------ |
| Docker multi stage | 14s     | 124MB | Docker-Image |
| BuildKit           | 15s     | 124MB | Docker-Image |
| Buildpacks         | 1min30s | 802MB | Docker-Image |
| Jib                | 16s     | 146MB | OCI-Image-v1 |
|                    |         |       |              |

构建脚本

```shell
#!/bin/bash

# 自定义gradle缓存镜像(原生docker构建需要此基础镜像加速构建)
cache_image_name="gradle-cache:latest"

# 远程构建的git仓库上下文
gitRepo="https://github.com/Maple-mxf/microservices-backend"

# 需要构建的模块
module="$1"

# 构建方式，可选有 docker|buildpakcks|jib|buildkit
buildType="$2"

# 构建的镜像版本
version=$(date +'%Y%m%d%H%M%S')

# 远程镜像仓库
remoteCR="ccr.ccs.tencentyun.com/tcb-2446011668-egvs/"

echo "module is $module"

if [ -z "$module" ];then
    echo "module args require"
    exit 0
fi
if [ -z "$buildType" ];then
    echo "default buildType is docker"
    buildType="docker"
fi

# docker images filter cmd https://docs.docker.com/engine/reference/commandline/images/
if [[ -n $(docker images -q --filter=reference=$cache_image_name) ]];then
    echo "Found gradle cache images"
else
  echo "Not Found gradle cache images. process docker build"
  docker build \
  -f Dockerfile.cache \
  -t ${cache_image_name} ${gitRepo}
fi

# 构建镜像
timer_start=$(date "+%Y-%m-%d %H:%M:%S")

case $buildType in
"docker")
docker build \
--no-cache  \
--build-arg MODULE="${module}" \
--build-arg CACHE_IMAGE="${cache_image_name}" \
--tag "${module}":"${version}" ${gitRepo} \
&& docker image prune -f
;;

"buildpacks")
pack build --path=./ --builder heroku/buildpacks:20 "${module}":"${version}"
;;

"buildkit")
docker build \
--no-cache  \
--build-arg MODULE="${module}" \
--build-arg CACHE_IMAGE="${cache_image_name}" \
--file Dockerfile.buildkit \
--tag "${module}":"${version}" ${gitRepo}
;;

"jib")
gradle :"${module}":jib --image=$remoteCR"${module}":"${version}"
;;
esac

timer_end=$(date "+%Y-%m-%d %H:%M:%S")
duration=$(echo $(($(date +%s -d "${timer_end}") - $(date +%s -d "${timer_start}"))) | awk '{t=split("60 s 60 m 24 h 999 d",a);for(n=1;n<t;n+=2){if($1==0)break;s=$1%a[n]a[n+1]s;$1=int($1/a[n])}print s}')

echo "Build success, buildType=$buildType,costTime=$duration"
```

下载远程构建脚本并且构建本文示例的gradle demo项目

1. ```shell
   curl https://raw.githubusercontent.com/apache/apisix/master/utils/install-dependencies.sh -sL | bash -
   ```

TODO：

1. Buildpacks优化（cache-image和volume）
2. Jib优化
3. OCI 和 Docker格式的镜像区分
