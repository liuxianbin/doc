---
title: EMQX部署
date: 2026-09-30
tags:
---

### 开源版本

**5.8.x** 系列是传统意义上的开源版，完全支持集群，且无需任何许可证

### 下载

```
https://github.com/emqx/emqx/releases/download/v5.8.9/emqx-5.8.9-el7-amd64.rpm
```

### 安装

EMQX 依赖 Erlang 运行时，直接用 rpm -ivh 容易因依赖报错。推荐用 yum localinstall，它会自动处理依赖

```bash
# 安装
sudo yum localinstall emqx-5.8.9-1.el7.x86_64.rpm
```

### 启动

```bash
# 启动
sudo systemctl start emqx
# 设置开机自启
sudo systemctl enable emqx
# 查看状态
sudo systemctl status emqx
```

### 放行防火墙端口

CentOS 7 默认开启 firewalld，如果不放行端口，Dashboard 和 MQTT 客户端都连不上。EMQX 主要用到这几个端口：

```bash
sudo firewall-cmd --permanent --add-port=1883/tcp # MQTT 标准端口
sudo firewall-cmd --permanent --add-port=8883/tcp # MQTT/TLS 加密端口
sudo firewall-cmd --permanent --add-port=8083/tcp # MQTT/WebSocket 端口
sudo firewall-cmd --permanent --add-port=18083/tcp # Dashboard 管理控制台

sudo firewall-cmd --reload
```

### 访问 Dashboard

如果端口正常监听，就可以打开浏览器访问 Dashboard：

- http://127.0.0.1:18083
- 默认用户名：admin
- 默认密码：public

### 集群

假设你有两个节点：

- 节点 1：192.168.0.101
- 节点 2：192.168.0.102

在节点 1（192.168.0.101）上修改

```hocon
node {
  name = "emqx@192.168.0.101" # 改成实际的节点 IP
  cookie = "emqxsecretcookie" # 两个节点必须完全一致
  data_dir = "/var/lib/emqx"
}

cluster {
  name = emqxcl
  discovery_strategy = static # 从 manual 改为 static
  static {
    seeds = ["emqx@192.168.0.101", "emqx@192.168.0.102"] # 列出所有节点
  }
}
```

在节点 2（192.168.0.102）上修改

```hocon
node {
  name = "emqx@192.168.0.102" # 唯一区别在这里
  cookie = "emqxsecretcookie"
  data_dir = "/var/lib/emqx"
}

cluster {
  name = emqxcl
  discovery_strategy = static
  static {
    seeds = ["emqx@192.168.0.101", "emqx@192.168.0.102"] # 与节点1保持一致
  }
}
```

配置改完后，在每个节点上重启服务：

```bash
sudo systemctl restart emqx
```

### 开放集群通信端口

这是最容易出错的地方。EMQX 5.x 集群需要开放特定的端口，端口号与节点名中的数字后缀有关：

集群发现端口：基础端口 4370 + 节点名中的数字后缀。例如，emqx@192.168.0.101 没有数字后缀，偏移量为 0，端口是 4370；如果是 emqx1，端口就是 4371。

集群 RPC 端口：基础端口 5370 + 数字后缀。对应地，分别是 5370 或 5371。

你需要为每个节点的防火墙放行对应的端口。以节点名 emqx@192.168.0.101 为例：

```bash
sudo firewall-cmd --permanent --add-port=4370/tcp
sudo firewall-cmd --permanent --add-port=5370/tcp
sudo firewall-cmd --reload
```

### 启动节点并加入集群

在节点2上执行，加入节点1

```sh
sudo emqx ctl cluster join emqx@192.168.0.101
```

### 验证集群状态

在任意节点执行：

```bash
sudo emqx ctl cluster status
```

### 常见问题

1、**密码错误**
遇到的密码错误问题，大概率是因为修改 node.name 后，EMQX 的数据存储目录“漂移”了，导致 Dashboard 用户数据看起来像是被“重置”了
**解决办法**
用命令行重置密码，这是最快的恢复方式

```bash
 emqx_ctl admins passwd admin 123456
```
