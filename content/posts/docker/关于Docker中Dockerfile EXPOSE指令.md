---
title: Docker EXPOSE指令

date: 2021-04-06 17:25:17

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Docker' , 'Dockerfile' ]

categories: [ 'Docker' ]
---

> 原由: 在更新 [alibaba-sentinel](https://github.com/alibaba/Sentinel),发布版本的时候，查看 [docker/issues](https://github.com/alibaba/Sentinel/issues/267) 在个人推荐中发现了一个[暴露csp.sentinel.api.port](https://github.com/anjia0532/sentinel-docker/issues)

## [Dockerfile](https://github.com/anjia0532/sentinel-docker/blob/master/Dockerfile)

```docker
FROM openjdk:8-jdk-alpine

ARG SENTINEL_VERSION="1.6.2"

WORKDIR /home/sentinel

RUN adduser -S sentinel && \
    apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" >  /etc/timezone && \
    rm -rf /var/cache/apk/* && \
    sed -i "s/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g" /etc/apk/repositories && \
    wget -O sentinel-dashboard.jar "https://github.com/alibaba/Sentinel/releases/download/${SENTINEL_VERSION}/sentinel-dashboard-${SENTINEL_VERSION}.jar"

USER sentinel

EXPOSE 8080
CMD java ${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom -jar "/home/sentinel/sentinel-dashboard.jar"
```

和我写的[docker](https://github.com/hb0730/docker)写的差不多，也只暴露出了 `8080`端口，因此特地查看了[docs](https://docs.docker.com/engine/reference/builder/#expose)

## expose 指令

原文:  The `EXPOSE` instruction informs Docker that the container listens on the specified network ports at runtime. You can specify whether the port listens on TCP or UDP, and the default is TCP if the protocol is not specified.

The `EXPOSE` instruction does not actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs the container, about which ports are intended to be published. To actually publish the port when running the container, use the `-p` flag on `docker run` to publish and map one or more ports, or the `-P` flag to publish all exposed ports and map them to high-order ports.

By default, `EXPOSE` assumes TCP. You can also specify UDP:

也就是说 **`EXPOSE`实际上并未布端口。它充当构建映像的人员和运行容器的人员之间的一种文档类型，有关打算发布哪些端口的信息**，如果需要发行端口，需要宿主机访问可以使用`docker run -p`去**指定映射端口**,

## docker-compose

在`docker-compose`中 `expose`大致和`Dockerfile`一致

> Expose ports without publishing them to the host machine - they’ll only be accessible to linked services. Only the internal port can be specified. [expose](https://docs.docker.com/compose/compose-file/compose-file-v3/#expose) 

> 公开端口，只有`link`链接的服务才能访问

如果**需要公开端口并映射到宿主机上**我们应使用`ports`指令

# 参考

* [Dockerfile#expose](https://docs.docker.com/engine/reference/builder/#expose)
* [docker-compose#expose](https://docs.docker.com/compose/compose-file/compose-file-v3/#expose) 
* [docker-compose#ports](https://docs.docker.com/compose/compose-file/compose-file-v3/#ports)
