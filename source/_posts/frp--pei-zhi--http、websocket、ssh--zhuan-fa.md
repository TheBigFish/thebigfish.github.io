---
title: frp 配置 http、websocket、ssh 转发
date: 2018-11-07 14:37:56
updated: 2018-11-07 14:37:56
tags: []
author: TheBigFish
---
# frp 配置 http、websocket、ssh 转发

参考 [frp#75](https://github.com/fatedier/frp/issues/75)

## http 不使用域名转发

`frps.ini`

```ini
[common]
bind_port = 7000
```

`frpc.ini`

```ini
[common]
server_addr = aaa.bbb.ccc.ddd
server_port = 7000

[tcp_port]
type = tcp
local_ip = 127.0.0.1
local_port = 2333
remote_port = 3333
```

在外网通过 <http://aaa.bbb.ccc.ddd:3333> 访问到内网机器里的 <http://127.0.0.1:2333> 了

## ssh 转发

`frpc.ini`

```ini
[common]
server_addr = aaa.bbb.ccc.ddd
server_port = 7000

[ssh]
type = tcp
local_ip = 192.168.0.1
local_port = 22
remote_port = 7022

[tcp_port]
type = tcp
local_ip = 192.168.0.1
local_port = 8888
remote_port = 8888
```

在外网 `ssh` 通过 `ssh -oPort=7022 user@aaa.bbb.ccc.ddd` 访问内网机器

在外网 `http` 通过 <http://aaa.bbb.ccc.ddd:8888> 访问到内网机器里的 <http://127.0.0.1:8888> 了

通过 `ws://aaa.bbb.ccc.ddd:8888` 访问 websocket

## 运行服务

`nohup ./frps -c ./frps.ini &`


***
Sync From: https://github.com/TheBigFish/blog/issues/3
