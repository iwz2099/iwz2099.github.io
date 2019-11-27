# 基于Docker-compose搭建jenkins


#### docker-compose配置

```
version: '2'
 
services:
  jenkins:
    image: jenkins/jenkins:latest
    restart: always
    environment:
      JAVA_OPTS: "-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai -Djava.awt.headless=true -Dmail.smtp.starttls.enable=true"
    ports:
      - "80:8080"
      - "50000:50000"
    volumes:
      - '/ssd/jenkins:/var/jenkins_home'
      - '/var/run/docker.sock:/var/run/docker.sock'
      - '/etc/localtime:/etc/localtime:ro'
    dns: 223.5.5.5
    networks:
      - extnetwork
networks:
   extnetwork:
      ipam:
         config:
         - subnet: 172.255.0.0/16
```

#### 启动服务
```
docker-compose  up -d 
```
