---
layout:     post
title:      "docker常用命令"
subtitle:   "docker"
date:       2026-02-06 20:40:00
author:     "vs code"
catalog: false
published: true
header-style: text
tags:
  - docker
---

sudo docker run --privileged=true --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
-itd --name llvm \
-p 30023:22 -p 4455:2222 \
--restart always \
-v /root/llvm:/workspace \
-w /workspace \
ubuntu-llvm:v1.1 \
/bin/bash

docker images 查看系统内的所有镜像
docker ps  查看正在运行的容器

docker commit docker_id image_name:image_version  将容器中的修改提交到镜像中

导出镜像 (docker save)：镜像（包含所有层、历史记录和标签）导出
基本语法： docker save <镜像名>:<标签> | gzip > <文件名>.tar.gz
docker save nginx:latest | gzip > nginx_image.tar.gz
导入镜像文件对应 save 的镜像： 
docker load < nginx_image.tar.gz (注意：load 会自动处理解压)


导出容器 (docker export)
如果你想把一个正在运行或已停止的容器（仅快照，不保留镜像历史和层）导出，请使用 docker export。
基本语法： docker export <容器ID或名称> | gzip > <文件名>.tar.gz
docker export my_web_container | gzip > container_snapshot.tar.gz
导入容器文件对应 export 的容器： 
gunzip -c container_snapshot.tar.gz | docker import - my_new_image:tag

docker stop container_id
docker rm container_id

一键清理 (Prune)删除所有状态为 exited（退出）的容器。
docker container prune
较旧版本：
docker rm $(docker ps -a -q -f status=exited)

删除所有容器（无论状态）：
docker rm -f $(docker ps -aq)

清理所有已停止的容器，所有未被使用的网络，所有虚悬镜像（即没有标签且未被引用的镜像层）。
docker system prune
