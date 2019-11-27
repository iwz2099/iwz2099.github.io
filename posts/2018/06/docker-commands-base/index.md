# Docker基础命令


#### 基础命令
```
docker info     #查看本地docker信息
docker search  openresty   #搜索远程镜像仓库
docker images      #查看当前系统镜像仓库镜像
docker pull centos  #获取远程镜像，默认不指定tag,为latest     
docker container  run -p 8000:80 --rm -t -i centos:latest /bin/bash  #-p将容器端口映射80到节点node8000端口，--rm终止后删除容器适合临时调试，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， -i 则让容器的标准输入保持打开
docker container exec -i -t b5d7bad57561 /bin/bash #进入正在运行容器
docker rmi $(docker images -f "dangling=true" -q) #批量删除无效的none镜像
```
<!--more-->

#### 导入导出镜像
```
docker save -o centos7.tar centos  #导出镜像
docker load < centos7.tar   #导入镜像(标准流载入，包括原数据如标签)
docker load  --input  centos7.tar   #从tar包中读取镜像（非标准载入）
```

#### 导入导出处容器
```
docker ps   #默认显示当前正在运行中的container
docker ps -a   #查看包括已经停止的所有容器
docker ps -l   #显示最新启动的一个容器（包括已停止的）

docker export 5a80afa126ba > centos7.5a80afa126ba.tar  #导出容器快照
docker import 5a80afa126ba > centos7.5a80afa126ba.tar  #从容器快照文件导入到镜像仓库
```

注：用户既可以使用 docker load 来导入镜像存储文件到本地镜像库，也可以使用 docker import 来导入一个容器快照到本地镜像库。这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。

#### 删除镜像或容器
```
docker rmi jdeathe/centos-ssh #删除本地镜像
or
docker image rm jdeathe/centos-ssh #删除本地镜像
docker rm dbd4d83097f5    #删除本地容器
or
docker container rm dbd4d83097f5 #删除本地容器
```

#### 关闭重启容器
```
docker container kill dbd4d83097f5  #关闭正在运行的容器
docker container restart dbd4d83097f5  #重启容器
```
#### 构建镜像
```
docker image build    .      #根据当前目录dockerfile构建镜像
docker image build   -t cm-navigation:v0.0.1 .  #根据当前目录dockerfile构建镜像，并打tag
```
