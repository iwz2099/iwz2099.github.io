# Dockerfile多阶段构建


Docker多阶段构建理解:

* 构建镜像需要有一个基础镜像,后续操作就会基于该基础镜像构建
* docker镜像文件里有层级概念,每执行一次RUN指令,镜像就会多一层,所以通过减少层级来减少镜像大小
* 多个from的时候,只有最后一个from的镜像才是镜像的根镜像

自己项目部署中多阶段构建示例:
```
FROM golang:1.12.7 as build
MAINTAINER wanzi <iwz2099@163.com>
 
# 编译配置相关
ARG NAME=gaia
ARG FLAGS=-tags=jsoniter
ARG GOOS=linux
ARG GOARCH=amd64
ARG PORT_TO_EXPOSE=10020
 
ENV GOPROXY https://mirrors.aliyun.com/goproxy/
ENV GO111MODULE on
 
WORKDIR /opt/gaia
COPY . .
RUN GOOS=$GOOS GOARCH=$GOARCH go build -mod vendor -ldflags="-s -w" -o $NAME $FLAGS
 
FROM alpine
WORKDIR /opt/gaia
COPY --from=build /opt/gaia/gaia .
RUN mkdir -p /opt/gaia/conf
VOLUME ["/opt/gaia/conf"]
 
CMD ["./gaia"]
EXPOSE $PORT_TO_EXPOSE
```
