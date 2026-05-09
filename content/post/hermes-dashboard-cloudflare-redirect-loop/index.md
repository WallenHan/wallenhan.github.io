+++
date = '2026-05-09T22:27:24+08:00'
draft = false
title = 'Hermes Dashboard 反代访问异常排障复盘'
categories = ['ServerTech']
tags = ['Hermes', 'Cloudflare', 'OpenResty']
+++

# Hermes Dashboard 反代访问异常排障复盘

有些运维问题最麻烦的地方，不是它完全不会报错，而是它每一层看起来都对，但整体就是不工作。

这次我们做 Hermes Dashboard 的域名访问时，遇到的就是这样一个问题。

目标本来很简单：把 Hermes Dashboard 跑在服务器上，通过 `ai.syhan.dpdns.org` 访问，并且不要把管理端口直接暴露在公网。  
听起来像是一条非常标准的链路：

> 域名 → 反向代理 → 本机服务

但真正开始之后，事情比想象中绕得多。

## 一开始：服务其实是好的

Hermes Dashboard 本身并没有问题。  
我们先确认了它的监听状态，`9119` 端口确实起来了，服务也能正常响应请求。

后来为了安全起见，我们把它从对外监听收回到本机：

```bash
hermes dashboard --host 127.0.0.1 --no-open
```

这样做的好处是：

- 9119 不再直接暴露公网
- 只能由本机反向代理访问
- Dashboard 更符合生产环境的安全要求

真正适合生产的方式，应该是让它只在本机监听，然后由 OpenResty 或 Nginx 统一做代理。

## 第二层：OpenResty 反代看起来也没错

1Panel 管理下的 OpenResty 站点配置如下：

- `/opt/1panel/www/conf.d/ai..conf`
- `/opt/1panel/www/sites/ai./proxy/root.conf`

核心反代逻辑是：

```nginx
proxy_pass http://127.0.0.1:9119;
```

也就是说，外部请求应该经过：

```text
域名 -> OpenResty -> 127.0.0.1:9119 -> Hermes Dashboard
```

从本机回源测试看，这条链路是通的。

为了把入口理顺，我们还补了这些东西：

- `80 -> 301 https`
- `443 -> 反代到 9119`
- 一度加过 basic auth
- 后来又把 basic auth 去掉
- 再后来又重新加回去做登录保护
- 最后又再次关掉，确认问题不在认证层

一路排下来，你会发现：OpenResty 本身并不是坏的，反代也不是坏的。

## 真正让人迷惑的，是“浏览器报重定向次数太多”

在实际访问域名的时候，事情就开始变得诡异了。

不是 404，不是 500，也不是证书报错，而是浏览器直接提示：

> 重定向次数太多

这类现象通常不是后端完全不可用，而是某一层在不断改写请求路径或协议，导致请求在不同端之间来回循环。

于是问题就从“单个服务是否正常”变成了“整条访问链路的语义是否一致”。

## 根因：Cloudflare 代理模式与源站 HTTPS 重定向冲突

### 1. 域名流量并不是直接打到源站

我们检查 DNS 后发现，`ai.syhan.dpdns.org` 解析出来的是 Cloudflare 的边缘节点地址，而不是源站机器 IP。

这意味着实际链路是：

```text
浏览器 -> Cloudflare -> 源站
```

而不是直接访问服务器。

### 2. 源站本身会把 HTTP 重定向到 HTTPS

在 1Panel 的 OpenResty 站点里，我们配置了：

- `80 -> 301 https`
- `443 -> 反代到 9119`

这是一种很常见、也很合理的生产策略：所有明文 HTTP 请求统一升级到 HTTPS。

### 3. Cloudflare Flexible 模式会让语义发生冲突

如果 Cloudflare 使用的是 **Flexible** 模式，那么它的行为是：

- 用户访问 Cloudflare：HTTPS
- Cloudflare 到源站：HTTP

这时候就出现了一个非常关键的问题：

#### 源站视角
源站看到的是 **HTTP 请求**，于是按照配置把它重定向到 HTTPS。

#### Cloudflare 视角
Cloudflare 仍然坚持用 HTTP 回源。

于是双方开始互相“纠正”：

1. 浏览器访问 `https://ai.syhan.dpdns.org`
2. 请求进入 Cloudflare
3. Cloudflare 以 HTTP 回源到源站
4. 源站把 HTTP 重定向到 HTTPS
5. Cloudflare 再按自己的策略回源
6. 浏览器最终陷入 **重定向循环**

这就是“重定向次数太多”的根本原因。

## 为什么关闭 Cloudflare 代理模式后问题就好了

当我们把 Cloudflare 的代理模式关闭，也就是改成 **DNS only** 后，域名访问立刻恢复正常。

这说明什么？

说明：

- Hermes Dashboard 是正常的
- OpenResty 反代是正常的
- basic auth 不是根因
- 真正的问题在 Cloudflare 代理层与源站 HTTPS 策略的冲突

也就是说，问题不是“服务没起来”，而是“外层代理和源站对协议语义理解不一致”。

## 为什么推荐 Full (strict)

如果未来需要重新打开 Cloudflare 代理模式，正确的方式不是 Flexible，而是：

- **Full**
- 更推荐 **Full (strict)**

### Full (strict) 的含义

它表示：

- 浏览器到 Cloudflare：HTTPS
- Cloudflare 到源站：HTTPS
- Cloudflare 会验证源站证书是否有效

### 为什么它适合这个场景

因为我们的源站本来就希望：

- HTTP 自动升级到 HTTPS
- 正式入口走 HTTPS

如果 Cloudflare 回源也使用 HTTPS，那么双方语义一致，重定向循环就不会再发生。

换句话说：

- Flexible：Cloudflare 用 HTTP 回源，容易和源站的 HTTP→HTTPS 冲突
- Full (strict)：Cloudflare 也用 HTTPS 回源，和源站策略一致

所以在这种架构中，**Full (strict) 是更合理的生产选择**。

## 最终生产配置建议

经过完整排查后，比较稳妥的生产方案是：

### 1. Hermes Dashboard 只监听本机

```bash
hermes dashboard --host 127.0.0.1 --no-open
```

### 2. OpenResty 统一对外入口

- `80 -> 443`
- `443 -> 127.0.0.1:9119`

### 3. 关闭或谨慎使用 basic auth

- 如果要保留认证，可以作为额外安全层
- 如果正在排障，最好先去掉认证，避免干扰判断

### 4. Cloudflare 代理模式要么关闭，要么用 Full (strict)

- 排障阶段：优先 DNS only
- 生产阶段：如果要用代理，建议 Full (strict)

## 经验总结

这次问题最重要的经验，不是某一行配置，而是对整条链路的理解。

### 1. 单点没问题，不代表整体没问题
Dashboard 能起、Nginx 能配、证书也正常，并不意味着整体链路一定正常。  
代理层、回源层、重定向策略叠加起来，才是最终用户看到的行为。

### 2. Cloudflare 不是完全透明的
一旦开启代理模式，Cloudflare 不只是 DNS，它还会参与：

- SSL 终止
- 回源协议
- 重定向行为
- 缓存与规则处理

### 3. Flexible 和强制 HTTPS 的源站容易冲突
只要源站坚持 `HTTP -> HTTPS`，Cloudflare 用 HTTP 回源就很容易出现重定向循环。

### 4. Full (strict) 才是和生产源站更一致的做法
它保证了外层代理和源站在协议上是统一的，不会互相打架。

## 结论

这次 Hermes Dashboard 访问异常的根因，不是反代配置写错，也不是 Dashboard 服务本身异常，而是：

> Cloudflare 的代理模式与源站的 HTTP→HTTPS 重定向策略发生了语义冲突，最终导致浏览器进入重定向循环。

当 Cloudflare 关闭代理模式后，问题立刻消失，说明源站链路本身没有问题。  
如果未来要恢复 Cloudflare 代理，应该使用 **Full (strict)**，让回源语义与源站策略保持一致。
