# Docker镜像最佳实践

## 5条最佳建议

### 1.仅安装产线需要依赖与软件

镜像尽可能最小原则

- 仅复制jar/war
- 使用自定义JRE(Java Runtime Environment)

### 2.使用多阶段构建

```
FROM maven:3.6.3-jdk-11-slim AS build
RUN mkdir /project
COPY . /project
WORKDIR /project
RUN mvn clean package -DskipTests
 
 
FROM adoptopenjdk/openjdk11:jre-11.0.9.1_1-alpine
RUN mkdir /app
COPY --from=build /project/target/java-application.jar /app/java-application.jar
WORKDIR /app
CMD "java" "-jar" "java-application.jar"
```

### 3.不要使用root运行应用

为了应用的安全，不要使用root账户运行应用

```
# Add app user
ARG APPLICATION_USER=appuser
RUN adduser --no-create-home -u 1000 -D $APPLICATION_USER

# Configure working directory
RUN mkdir /app && \
    chown -R $APPLICATION_USER /app

USER 1000
COPY --chown=1000:1000 ./app.jar /app/app.jar
WORKDIR /app

EXPOSE 8080
ENTRYPOINT [ "exec","/jre/bin/java", "-jar", "/app/app.jar" ]
```

### 4.尽可能处理终止信号量

推荐使用`exec`命令

```
ENTRYPOINT [ "exec","/jre/bin/java", "-jar", "/app/app.jar" ]
```

推荐处理应用终止回调方法

```
Runtime.getRuntime().addShutdownHook(yourShutdownThread);
```

### 5.使用.dockerigore,使用新版jdk

使用`.dockerignore`排除不需要的文件

使用`Java 8 update 191+ `和`jdk10+`版本，其他版本无法感知docker内存和cpu限制。

## 最小镜像实践

在云环境下，精简镜像有极大的资源利用率的提升，节省磁盘和网络带宽，有着极大的收益。

jdk1.8以下推荐使用`alpine`镜像

```
docker pull openjdk:8u212-jre-alpine3.9
```

jdk11+，由于不再提供jre运行时，推进使用`alpine`+jlink剪裁出最佳运行时。

### 自定义jre

**首先使用`jdeps`查看依赖的module**

```
$ jdeps --print-module-deps --ignore-missing-deps --recursive --multi-release 17 --module-path="./app/BOOT-INF/lib/*" --class-path ./app/BOOT-INF/lib/* app.jar
java.base,java.compiler,java.desktop,java.instrument,java.management,java.naming,java.prefs,java.scripting
```

对于spring 应用推荐增加`--ignore-missing-deps`，可以忽略spring 的一些module缺失，一般情况下我们的jar会包含spring所有依赖，对于这种fat.jar我们可以先解压该包，然后增加`--module-path="./app/BOOT-INF/lib/*" --class-path ./app/BOOT-INF/lib/*` 参数进行分析。

**最后使用`jlink`输出自定义jre**

```
$ jlink \
         --add-modules java.base,java.compiler,java.desktop,java.instrument,java.management,java.naming,java.prefs,java.scripting \
         --strip-debug \
         --no-man-pages \
         --no-header-files \
         --compress=2 \
         --output ./javaruntime
```



### 最佳镜像

以下为最佳实践案例

```
# base image to build a JRE
FROM amazoncorretto:17.0.3-alpine as corretto-jdk

# required for strip-debug to work
RUN apk add --no-cache binutils

# Build small JRE image
RUN $JAVA_HOME/bin/jlink \
    --verbose \
    --add-modules java.base,java.compiler,java.desktop,java.instrument,java.management,java.naming,java.prefs,java.scripting \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /customjre

# main app image
FROM alpine:latest
ENV JAVA_HOME=/jre
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# copy JRE from the base image
COPY --from=corretto-jdk /customjre $JAVA_HOME

# Add app user
ARG APPLICATION_USER=appuser
RUN adduser --no-create-home -u 1000 -D $APPLICATION_USER

# Configure working directory
RUN mkdir /app && \
    chown -R $APPLICATION_USER /app

USER 1000

COPY --chown=1000:1000 ./app.jar /app/app.jar
WORKDIR /app

EXPOSE 8080
ENTRYPOINT [ "exec","/jre/bin/java", "-jar", "/app/app.jar" ]
```

## Native Image

//TODO

## 参考

1. [How to reduce JVM docker image size](https://blog.wolt.com/engineering/2022/05/13/how-to-reduce-jvm-docker-image-size/)
2. [10 best practices to build a Java container with Docker](https://snyk.io/blog/best-practices-to-build-java-containers-with-docker/)
3. [使用 jlink 的 Java 运行时](https://learn.microsoft.com/zh-cn/java/openjdk/java-jlink-runtimes)