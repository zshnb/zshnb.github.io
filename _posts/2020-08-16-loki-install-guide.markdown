---
layout: post
title: loki安装踩坑记
date: 2020-08-16 13:50
author: zsh
catalog: true
tags:
    - loki
---

## 介绍

loki是grafana推出的一款轻量级日志收集框架，由于实在不想搭建ELK，于是决定尝试一下。鉴于网上现有文档资源较少，于是想在这里记录一下自己搭建过程中的踩坑过程，希望能帮到后面的同学。

### 步骤

#### 安装

官方文档上提供了4种安装方式，个人推荐使用docker或者docker-compose的方式安装，既简单又便捷。

- 下载官方docker-compose.yaml

  `wget https://raw.githubusercontent.com/grafana/loki/v1.6.0/production/docker-compose.yaml -O docker-compose.yaml`

- 修改docker-compose.yaml

  ```yaml
  version: "3"
  
  networks:
    loki:
  
  services:
    loki:
      image: grafana/loki:1.5.0
      ports:
        - "3100:3100"
      command: -config.file=/etc/loki/local-config.yaml
      networks:
        - loki
  
    promtail:
      image: grafana/promtail:1.5.0
      volumes:
        - /var/logs:/var/log // 挂载宿主机上的日志目录到promtail容器里
        - /etc/promtail/docker-config.yaml:/etc/promtail/docker-config.yaml // 使用外部配置文件覆盖默认配置文件
      command: -config.file=/etc/promtail/docker-config.yaml
      networks:
        - loki
  
    grafana:
      image: grafana/grafana:latest
      ports:
        - "3000:3000"
      networks:
        - loki
  
  ```

  

- 创建promtail的配置文件，路径就放在上面外部配置指定的路径里，promtail用来将容器日志发送到 Loki 或者 Grafana 服务上的日志收集工具，该工具主要包括发现采集目标以及给日志流添加上 Label 标签 然后发送给 Loki。

  ```yaml
  server:
    http_listen_port: 9080
    grpc_listen_port: 0
  
  positions:
    filename: /tmp/positions.yaml
  
  client:
    url: http://loki:3100/loki/api/v1/push
  
  scrape_configs:
  - job_name: qiangdong
    static_configs:
    - targets:
        - localhost
      labels:
        job: dev
        __path__: /var/log/**/*.log // promtail容器里日志的路径
  ```

- 执行`docker-compose up -d`启动服务

#### 启动

1. 访问`localhost:3000`，默认帐号密码是admin
2. 添加loki数据源
3. 点击左侧explore查看日志，选择配置文件里配置的jobs