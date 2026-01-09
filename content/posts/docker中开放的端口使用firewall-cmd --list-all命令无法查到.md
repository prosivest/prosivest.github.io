---
date : 2026-01-09T09:33:37+08:00
lastmod : 2026-01-09T09:33:37+08:00
title : 'Docker中开放的端口使用firewall cmd  --list all命令无法查到'
description : "" # 文章描述，与搜索优化相关
author : ["想上天的鱼"]
tags: ["docker","防火墙","端口"]
ShowToc: true
draft : false
---

> 参考chatgpt 5.2


**因为它只列出 Firewalld 自己管理的规则，而不显示其他程序（如 Docker）直接插入到内核 iptables 表中的规则。**


# 如何正确查看到这些端口规则？

虽然 `firewall-cmd` 查不到，但你可以通过以下几种底层方式验证端口是否在系统层面被开放：

## 1. **直接查看 iptables 的 nat 表（最底层真相）**
Docker 的端口映射主要在 `nat` 表中做 DNAT（目标地址转换）。你可以使用以下命令查看：
    
```bash
iptables -t nat -L -n | grep DOCKER
```
    
或者查看 Docker 专门的过滤链规则：
    
```bash
iptables -L DOCKER -n
```
    
## 2. **查看系统网络监听状态**
使用 `netstat` 或 `ss` 命令可以看到 Docker 代理进程（`docker-proxy`）是否正在监听宿主机的端口：
    
```bash
netstat -tulnp | grep docker
或者
ss -tulnp | grep docker

```
    
## 3. **使用 Docker 命令查看**
当然，最直接的方法是查询 Docker 自身的配置：
    
```bash
docker ps

```

输出中的 `PORTS` 一列会明确显示如 `0.0.0.0:8080->80/tcp` 的映射关系，这代表该端口已对宿主机所有网卡开放。
    

# 重要的安全隐患：Docker 穿透防火墙

这是一个非常重要且常被运维人员忽略的问题：**Docker 开放的端口默认会绕过 Firewalld 的限制**。

**场景示例**：
假设你在 Firewalld 中设置了“默认拒绝所有外部连接”，或者移除了所有服务的允许规则。此时，你启动了一个 `docker run -p 8080:80 ...` 的容器。
**结果**：外部网络**依然可以**直接访问宿主机的 8080 端口。

**原因**：
因为 Docker 修改的 iptables 规则（位于 `PREROUTING` 链）优先级非常高，数据包在进入 Firewalld 的常规过滤规则（通常在 `INPUT` 链或更靠后的位置）之前，就已经被 Docker 的规则捕获并转发进容器了。

# 如何解决这个安全控制问题？

如果你希望 Docker 的端口也能受防火墙策略控制（例如只允许特定 IP 访问），建议采用以下方案：

## 1. **方案一：在启动容器时绑定特定 IP（推荐）**
这是最简单且副作用最小的方法。不要使用默认的 `p 8080:80`（这意味着绑定 `0.0.0.0`，即所有 IP），而是显式绑定到回环地址：
    
```bash
docker run -p 127.0.0.1:8080:80 ...
```

这样端口只对宿主机内部开放。然后你可以利用 Nginx 在宿主机做反向代理，或者手动配置 Firewalld 的规则将特定流量转发给这个 localhost 端口。
    
## 2. **方案二：利用 DOCKER-USER 链（高级）**
Docker 官方预留了一个名为 `DOCKER-USER` 的 iptables 链，它位于 Docker 规则的最前面。Docker 每次重启网络时不会重置这个链。你可以在这里添加 iptables 规则来限制访问。
例如，只允许 `192.168.1.100` 访问容器网络：
    
```bash
iptables -I DOCKER-USER -i eth0 ! -s 192.168.1.100 -j DROP
```
    
## 3. **方案三：禁用 Docker 的 iptables 管理（慎用）**
修改 `/etc/docker/daemon.json`，添加配置 `"iptables": false`。
*注意：这会导致容器无法访问外网，且 `p` 端口映射功能会失效。你需要完全手动配置 iptables 的 NAT 和 Filter 规则，配置复杂度极高，不建议普通用户使用。*

# 总结

`firewall-cmd --list-all` 查不到 Docker 端口是因为 Docker **绕过了 Firewalld 的管理层**，直接操作了底层的 iptables `nat` 表。

要验证端口开放情况，请使用 `iptables -t nat -L` 或 `docker ps`。同时请务必警惕 Docker 这种机制带来的“防火墙穿透”效应，确保生产环境的安全策略不会因此失效。

