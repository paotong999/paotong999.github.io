## 一、docker hub使用
1、登录到docker hub：
```docker login {docker-hub}:8500/```
2、上传位置：
```{docker-hub}:8500/yingzi```
3、上传要求：相同 tag会覆盖

## 二、打包镜像-macOS
### 1. 登陆私有仓库
```
docker login {docker-hub}:8500
输入用户名和密码
```

### 2. 给本地镜像打标签（示例）
```
docker tag <本地镜像名>:<本地标签> {docker-hub}:8500/yingzi/<镜像名>:<目标标签>
# 例如
docker tag yingzi-fpf-tools:latest {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
```

### 3. 推送到仓库
```
docker push {docker-hub}:8500/yingzi/<镜像名>:<目标标签>
# 例如
docker push {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
```

### 4. 可选：直接构建并打上目标标签然后推送
```
docker build -t {docker-hub}:8500/yingzi/my-app:latest .
docker buildx build 
--platform linux/amd64,linux/arm64 
-t {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest 
--push 
.
docker push {docker-hub}:8500/yingzi/my-app:latest
```

## 三、拉取镜像并运行
```
# 拉取镜像
docker pull {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
# 查看本地镜像
docker images | grep yingzi-fpf-tools
# 后台运行（示例映射 8080 端口）
docker run -d --name yingzi-fpf-tools 
-p 5000:5000 
{docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
# 交互式运行（进入容器 shell）
docker run --rm -it {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest /bin/bash
# 查看最新日志
docker logs -f yingzi-fpf-tools
```

## 四、常用命令
```
停止容器: docker stop <容器名或ID>
删除容器（安全）: docker rm <容器名或ID>
强制删除容器（停止并删除）: docker rm -f <容器名或ID>
删除本地镜像: docker rmi <镜像名:标签或ID>
强制删除镜像（若被容器占用）: docker rmi -f <镜像名:标签或ID>
删除仓库标签镜像示例: docker rmi {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
批量删除所有容器: docker rm $(docker ps -a -q)（建议先 docker stop$(docker ps -a -q)）
批量删除所有镜像: docker rmi $(docker images -a -q) 或更安全用 docker image prune -a
删除未使用的卷/临时数据: docker volume rm <卷名> 或 docker volume prune
清理所有未使用资源（容器/网络/镜像/卷）: docker system prune -a --volumes（会删除很多内容，慎用）
```

## 五、疑难杂症
### 1. 报错：镜像架构不兼容
```
[root@HOSTNAME=172-19-102-42 yzadmin]$ docker pull {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
latest: Pulling from yingzi/yingzi-fpf-tools
no matching manifest for linux/amd64 in the manifest list entries
```
原因：
你在 macOS（可能是 M 系列芯片，arm64）上构建的镜像只包含了 arm64 架构，而 CentOS 7 服务器是 amd64 架构，因此无法拉取。
解决方案：
使用 Docker buildx 构建多架构镜像
```bash
# 1. 创建并使用新的 buildx builder（支持多平台）
docker buildx create --name multiarch-builder --use
# 2. 启动 builder 实例
docker buildx inspect --bootstrap
# 3. 登录私有仓库
docker login {docker-hub}:8500
# 4.1 构建并推送支持 linux/amd64 和 linux/arm64 的镜像
docker buildx build 
--platform linux/amd64,linux/arm64 
-t {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest 
--push 
.
# 4.2 构建仅推送支持 linux/amd64 的镜像
docker buildx build 
--platform linux/amd64 
-t {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest 
--push 
.
# 5. 查看镜像 manifest（确认包含 amd64）
docker buildx imagetools inspect {docker-hub}:8500/yingzi/yingzi-fpf-tools:latest
```