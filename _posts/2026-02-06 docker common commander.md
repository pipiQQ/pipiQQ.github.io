
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

docker commit docker_id docker_name:docker_version

导出镜像 (docker save)：镜像（包含所有层、历史记录和标签）导出
基本语法： docker save <镜像名>:<标签> | gzip > <文件名>.tar.gz
示例：
Bash
docker save nginx:latest | gzip > nginx_image.tar.gz
导入镜像文件对应 save 的镜像： docker load < nginx_image.tar.gz (注意：load 会自动处理解压)


导出容器 (docker export)
如果你想把一个正在运行或已停止的容器（仅快照，不保留镜像历史和层）导出，请使用 docker export。
基本语法： docker export <容器ID或名称> | gzip > <文件名>.tar.gz
示例：
Bash
docker export my_web_container | gzip > container_snapshot.tar.gz
导入容器文件对应 export 的容器： gunzip -c container_snapshot.tar.gz | docker import - my_new_image:tag

