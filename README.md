# gitlab-runner-CICD-sample
部署docker镜像 gitlab-runner集成部署CICD教程

使用了3台机器，

1- 找1台独立的运行runner的机器，便于后面举例比如IP是192.168.199.100，在这台机器上安装gitlab-runner
2- 注册runner到gitlab主机，便于后面举例比如gitlab安装在IP是172.0.0.188，
3- 部署docker镜像的目标主机，便于后面举例比如IP是10.1.1.99.

## 第一阶段：安装runner （在192.168.199.100机器上）
在IP=100机器上，docker安装 gitlab-runner，并注册到gitlab主机上，
直接命令运行：

```
docker run -d -m 1024m \
        --memory-reservation=512m \
        --name gitlab-runner --restart always \
        -v /opt/gitlab-runner/config:/etc/gitlab-runner \
        -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
```
这里将/opt/gitlab-runner/config目录挂载，默认配置文件config.toml 里面，就是gitlab-runner register
后生成的信息，

config.toml文件长得如下：
```
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "tencent_runner119" ##-----> 将来显示在gitlab网页上的runner名字
  url = "http://gitlab.****.cn/"   ##-----> gitlab主机的域名
  id = 1
  token = "N***************_BU" ##-----> 从gitlab 配置页面上取得
  token_obtained_at = 2022-10-11T08:58:11Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false   
    image = "docker:stable"
    memory = "1024m"
    memory_swap = "2048m"
    memory_reservation = "128m"
    privileged = false  ##-----> 非全部权限
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/root/.docker:/root/.docker", "/cache:/cache:rw", "/var/run/docker.sock:/var/run/docker.sock"]
    pull_policy = ["if-not-present"]
    shm_size = 0
```
这里主要注意的是目录挂载 volumes的配置。

因为我们的executor = "docker"是以docker形式运行，而注册到gitlab的上去之后，当gitlab的pipline管道任务执行时，
将是以docker-in-docker形式运行。

token取得从这里：
![image](https://user-images.githubusercontent.com/12711164/196850246-181416a0-942e-4bb6-91ad-61bf34559842.png)

这里的/root/.docker目录下放的TLS鉴权登录用的client端文件，这个稍后再说如何制作。
这里的/cache目录下放的.m2 maven打包用的文件依赖包，为了提速docker maven java build打包的速度，以及setting.xml配置文件。

## 第二阶段：部署的目标主机上的 docker TLS登录的证书制作（在目标主机10.1.1.99上）

下面这个脚本，要在 部署docker镜像的目标主机,IP=10.1.1.99机器上执行.
脚本中的IP 可以替换成 实际IP，如10.1.1.99，k8s_pods参数没有可以注释掉这一行。对应修改echo "subjectAltName = IP:$IP,IP:$k8s_pods,IP:127.0.0.1" >> extfile.cnf这一句。

在使用脚本前，先将一些必要的信息填写好，在空目录下执行即可，

```
#!/bin/bash
#
# -------------------------------------------------------------
# 自动创建 Docker TLS 证书
# -------------------------------------------------------------
# 以下是配置信息
# --[BEGIN]------------------------------

CODE="devops"
IP="***.***.***.***"
PASSWORD="****"
COUNTRY="CN"
STATE="shanghai"
CITY="shanghai"
ORGANIZATIONAL_UNIT="****"
ORGANIZATION="$ORGANIZATIONAL_UNIT.com"
COMMON_NAME="$IP"
EMAIL="****@*****.com"
# 这里添加你允许的地址段
k8s_pods="***.***.***.***/16"
# --[END]--

# Generate CA key
openssl genrsa -aes256 -passout "pass:$PASSWORD" -out "ca-key-$CODE.pem" 4096
# Generate CA
openssl req -new -x509 -days 365 -key "ca-key-$CODE.pem" -sha256 -out "ca-$CODE.pem" -passin "pass:$PASSWORD" -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORGANIZATION/OU=$ORGANIZATIONAL_UNIT/CN=$COMMON_NAME/emailAddress=$EMAIL"
# Generate Server key
openssl genrsa -out "server-key-$CODE.pem" 4096

# Generate Server Certs.
openssl req -subj "/CN=$COMMON_NAME" -sha256 -new -key "server-key-$CODE.pem" -out server.csr

echo "subjectAltName = IP:$IP,IP:$k8s_pods,IP:127.0.0.1" >> extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "server-cert-$CODE.pem" -extfile extfile.cnf

# Generate Client Certs.
rm -f extfile.cnf

openssl genrsa -out "key-$CODE.pem" 4096
openssl req -subj '/CN=client' -new -key "key-$CODE.pem" -out client.csr
echo extendedKeyUsage = clientAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "cert-$CODE.pem" -extfile extfile.cnf

rm -vf client.csr server.csr

chmod -v 0400 "ca-key-$CODE.pem" "key-$CODE.pem" "server-key-$CODE.pem"
chmod -v 0444 "ca-$CODE.pem" "server-cert-$CODE.pem" "cert-$CODE.pem"

# 打包客户端证书
mkdir -p "tls-client-certs-$CODE" && \
cp -f "ca-$CODE.pem" "cert-$CODE.pem" "key-$CODE.pem" "tls-client-certs-$CODE/" && \
cd "tls-client-certs-$CODE" && \
tar zcf "tls-client-certs-$CODE.tar.gz" * && \
mv "tls-client-certs-$CODE.tar.gz" ../ && \
cd .. && \
rm -rf "tls-client-certs-$CODE"

# 拷贝服务端证书
mkdir -p /etc/docker/certs.d
cp "ca-$CODE.pem" "server-cert-$CODE.pem" "server-key-$CODE.pem" /etc/docker/certs.d/

```

这个脚本执行完后，生成文件如下：
```
ca.srl：CA签发证书的序列号记录文件
ca-cert.pem：CA证书
ca-key.pem：CA密钥
server-key.pem：服务端密钥
server-req.csr：服务端证书签名请求文件
server-cert.pem：服务端证书
client-key.pem：客户端密钥
extfile.cnf：客户端证书扩展配置文件
client-req.csr：客户端证书签名请求文件
client-cert.pem：客户端证书
```
在/etc/docker/certs.d/目录下会生成3个server端用的文件：

```
-r--r--r-- 1 root root 2171 Oct 20 09:29 ca-svr.pem
-r--r--r-- 1 root root 1939 Oct 20 09:29 server-cert-svr.pem
-r-------- 1 root root 3243 Oct 20 09:29 server-key-svr.pem
```

然后，需要修改docker的守护进程配置文件，ubuntu系统下修改：

## vi /etc/docker/daemon.json

顺便确认下 docker 仓库国内镜像源。
```
{
  "tlsverify": true,
  "tlscert": "/etc/docker/certs.d/server-cert-svr.pem",
  "tlskey": "/etc/docker/certs.d/server-key-svr.pem",
  "tlscacert": "/etc/docker/certs.d/ca-svr.pem",
  "insecure-registries": [
    "nexus.***.cn:8001",
    "nexus.***.cn:8002"
  ],
  "registry-mirrors": [
       "https://mirror.ccs.tencentyun.com",
       "https://reg-mirror.qiniu.com",
       "https://registry.docker-cn.com"
  ]
}
```
此外还要开启docker的 2376 能外网TLS加密socket访问的端口，ubuntu系统通过修改

## vi  /lib/systemd/system/docker.service

```
# vim /usr/lib/systemd/system/docker.service
[Service]

ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H unix://var/run/docker.sock --containerd=/run/containerd/containerd.sock
```

让Docker Daemon把服务暴露在tcp的2375端口上，这样就可以在网络上操作Docker了。Docker本身没有身份认证的功能，只要网络上能访问到服务端口，
就可以远程操作Docker。

修改完后，重启docker服务。

```
systemctl daemon-reload
systemctl restart docker

```

还有一种修改 /etc/docker/daemon.json文件的方式，在ubuntu上修改后启动docker服务报错，有待调查。
```
vim /etc/docker/daemon.json

{
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
```
修改完后别忘了 restart docker 重启服务。查看开启端口情况
netstat -tunlp 确认端口监听是否启动。
![image](https://user-images.githubusercontent.com/12711164/196853250-a264895a-30fd-44eb-941d-4d61ae9b957c.png)

### 至此，待部署镜像的目标主机上的TLS证书配置和服务已经就绪OK。接下来去配置runner（client）端的TLS证书文件。

## 第三阶段 runner端 TLS配置文件 （在192.168.199.100机器上）

上面TLS证书脚本执行，在当前目录下还生成了个 tls-client-certs-$CODE.tar.gz 的文件，是客户端用的证书文件，先

scp tls-client-certs-$CODE.tar.gz root@远程主机IP:/root/.docker 复制到runner的宿主机192.168.199.100上。
解压后到/cert目录下，得到这3个文件备用：
```
-r--r--r-- 1 root root 2171 Oct 20 09:29 ca.pem
-r--r--r-- 1 root root 1903 Oct 20 09:29 cert.pem
-r-------- 1 root root 3243 Oct 20 09:29 key.pem
```

因为之前 runner的 config.toml文件中，已经配置了将/root/.docker目录映射到runner执行器docker中，
所以，后面我们在gitlab-ci.yml文件中，写远程部署语句就可以直接加载到tls登录用的client文件。

##  第四阶段 gitlab CI文件配置 （在172.0.0.188机器上）

至此，我们已经配置好了：

1-- 部署镜像的目标主机的TLS 2376端口访问
2-- runner启动挂载TLS client登录用的认证文件

此时我们可以在gitlab-ci文件中，通过远程登录到10.1.1.99主机区拉取镜像，实现镜像部署的更新。

```
image: docker:19.03.1

variables:
  MAVEN_CLI_OPTS: "-s /cache/.m2/settings.xml --batch-mode"
  MAVEN_OPTS: -Dmaven.repo.local=/cache/.m2
  DOCKER_REGISTRY: nexus.*****.cn    // docker镜像仓库主机
  #部署镜像的目标主机多服务器配置
  HOST_TEST: 10.1.1.99

stages:
  - build
  - test
  - deploy


#镜像构建 采用mvn jib方式 runner中的.m2依赖包 提速打包 
build:
  image: maven:3-jdk-8
  stage: build
  tags:
    - run-hwgz1
  script:
    - mvn clean package jib:build $MAVEN_CLI_OPTS -Dmaven.test.skip=true -Djib.baseImageCache=/cache/.m2/jib -X
    - (rm -r /cache/versions/$CI_PIPELINE_ID || true) && mkdir -p /cache/versions/$CI_PIPELINE_ID
    - echo $(mvn $MAVEN_CLI_OPTS -q -Dexec.executable=echo -Dexec.args='${project.version}' --non-recursive exec:exec) > /cache/versions/$CI_PIPELINE_ID/.version
    - cat /cache/versions/$CI_PIPELINE_ID/.version
  when: manual


#测试环境发布
deploy-to-gfy:
  stage: deploy
  tags:
    - run-hwgz1
  variables:
    #手动声明部署到指定的服务器
    DOCKER_HOST: tcp://$HOST_TEST:2376
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: /root/.docker/cert  
    # 这个目录是在gitlab-runner的配置文件config.toml中从宿主机器映射进来
    # 该目录下有 ca.pem  cert.pem  key.pem 这3个文件，文件名不要改！！ 
    # 参考：https://hub.docker.com/_/docker/#tls
    #       https://cloud.tencent.com/developer/article/1683689
    #       https://docs.gitlab.com/ee/ci/docker/using_docker_build.html
    GIT_STRATEGY: none
  script:
    - VERSION=$(cat /cache/versions/$CI_PIPELINE_ID/.version)
    #- ls /root/.docker/test
    #- echo ".m2 file list:"
    #- ls /cache/.m2
    # 直接docker login登录过去 开始拉取镜像，实现更新
    - docker login $DOCKER_REGISTRY
    - docker pull $DOCKER_REGISTRY/gfy-$CI_PROJECT_NAME:$VERSION
    - docker stop $CI_PROJECT_NAME || true
    - docker rm $CI_PROJECT_NAME || true
    #启动容器 连接至test-1网络
    - docker run -d -v /var/log/$CI_PROJECT_NAME:/var/log/$CI_PROJECT_NAME -p 9082:9082 --network prod --name $CI_PROJECT_NAME -e spring.profiles.active=prod $DOCKER_REGISTRY/gfy-$CI_PROJECT_NAME:$VERSION
#  only:
#    - /^release-(.*)$/
  when: manual




```



