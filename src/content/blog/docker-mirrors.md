---
author: robus
pubDatetime: 2024-11-19T15:22:00Z
modDatetime: 2024-11-19T15:22:00Z
title: Docker 镜像列表(目前国内可用)
slug: "112273"
featured: false
draft: false
tags:
  - docker
description:
  Docker image list.
--- 


```sh
{ 
"registry-mirrors" : 
[ 
    "https://docker.m.daocloud.io", 
    "https://docker.jianmuhub.com",
    "https://huecker.io",
    "https://dockerhub.timeweb.cloud",
    "https://dockerhub1.beget.com",
    "https://noohub.ru",
    "https://docker.923828.xyz",
    "https://docker.registry.cyou",
    "https://docker-cf.registry.cyou",
    "https://dockercf.jsdelivr.fyi",
    "https://docker.jsdelivr.fyi",
    "https://dockertest.jsdelivr.fyi",
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://dockerhub.azk8s.cn",
    "https://mirror.ccs.tencentyun.com",
    "https://registry.cn-hangzhou.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn"
] 
}
```

```sh
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://registry.docker-cn.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://hub-mirror.c.163.com",
        "https://mirror.baidubce.com",
        "https://registry.devops-engineer.com.cn",
        "https://docker.13140521.xyz",
        "https://ccr.ccs.tencentyun.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```