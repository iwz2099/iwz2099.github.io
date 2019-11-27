# 基于K8S部署gitlab-runner


#### 部署gitlab-runner

这里基于helm部署，参考：https://gitlab.com/gitlab-org/charts/gitlab-runner.git

```
helm install --namespace gitlab-managed-apps --name k8s-gitlab-runner -f  values.yaml 
```
注意：values.yaml文件需要设置privileged: true

#### 构建基础镜像(docker in docker)

Dockerfile文件内容：
```
FROM docker:19.03.1-dind
WORKDIR /opt
RUN echo "nameserver 114.114.114.114" >> /etc/resolv.conf
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk update
RUN apk upgrade
RUN apk add g++ gcc make docker docker-compose  git
```
构建镜像并推送到harbor
```
docker build -t registry.test.cn/devops/docker-tool:19.03.1  .
docker push  registry.test.cn/devops/docker-tool:19.03.1
```

#### Gitlab-CI测试：

.gitlab-ci.yml文件内容如下:
```
image: registry.test.cn/devops/docker-tool:19.03.1

variables:
  REPO_NAME: gitlab.test.cn/xxx/xxxx
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  
before_script:
  - mkdir -p $GOPATH/src/$(dirname $REPO_NAME)
  - ln -svf $CI_PROJECT_DIR $GOPATH/src/$REPO_NAME
  - cd $GOPATH/src/$REPO_NAME

stages:
  - deploy

deploy:
  tags:
    - k8s-gitlab-runner #指定runner
  only:
    - tags
  stage: deploy
  services:
    - registry.test.cn/devops/docker-tool:19.03.1
  script:
    - export DOCKER_HOST='tcp://localhost:2375'
    - docker login -u "$Harbor_bce_user" -p "$Harbor_ecs_passwd" $Harbor_ecs_address #gitlab后台设置统一变量
    - make deploy VERSION=$CI_COMMIT_REF_NAME #根据git tag构建镜像
```
