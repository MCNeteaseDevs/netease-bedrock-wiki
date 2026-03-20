---
front:
hard: 进阶
time: 30分钟
---

# 使用WaterdogPE搭建群组服

本文将指导您使用WaterdogPE搭建群组服务器，您需要具有群组服部署相关经验。WaterdogPE是一款适用于MCBE的群组服（反代）软件，类似MCJE的BungeeCord以及Velocity。
有关更多WaterdogPE的相关文档，您可以移步至其[官方文档站](https://docs.waterdog.dev/books/waterdogpe-setup)，本文将侧重于网易相关内容。

## 构建WaterdogPE

由于WaterdogPE官方仓库并未提供网易客户端支持，您需要使用[AllayMC的分叉版本](https://github.com/AllayMC/WaterdogPE)，请自行构建jar包。

## 启用网易支持

WaterdogPE启动后会生成一个`config.yml`配置文件，您需要修改以下配置项以启用网易客户端支持：

```yaml
netease_client_support: true
# Optional: only allow NetEase clients to connect
only_allow_netease_client: false
```

> 启用网易客户端支持后，所有RakNet v8客户端都将被视为网易客户端。

## 配置下游服务器

在`config.yml`中，您需要配置WaterdogPE所代理的下游Allay服务器。以下是一个示例配置：

```yaml
# 下游服务器列表
servers:
  lobby:
    address: 127.0.0.1:19133
    public_address: play.myserver.com:19133
  survival:
    address: 127.0.0.1:19134

# 玩家连接时的服务器优先级（玩家进服后默认进入列表中的第一个服务器）
priorities:
  - lobby
  - survival

# 强制域名映射（可选）
forced_hosts:
  lobby.myserver.com: lobby
```

其中：
- `address`：下游服务器的实际地址和端口
- `public_address`：（可选）公开地址，用于玩家直连该子服
- `priorities`：玩家进入代理后默认连接的服务器顺序

## 其他常用配置

以下是`config.yml`中一些值得关注的配置项：

```yaml
# 代理监听地址与端口
host: 0.0.0.0:19132
# 最大玩家数
max_players: 20

# 传递玩家真实信息到下游服务器（推荐启用）
use_login_extras: true

# 启用快速转服（玩家在子服之间转移时不会断开连接）
fast_transfer: true

# 压缩等级设置
# 上行（代理→客户端），值越高带宽越省但CPU占用越高
upstream_compression_level: 7
# 下行（代理→子服），本地网络可设为较低值
downstream_compression_level: -1
```

> `use_login_extras`启用后，代理会在LoginPacket中附加`Waterdog_IP`（玩家真实IP）和`Waterdog_XUID`（玩家XUID）等信息，方便下游服务器获取玩家的真实连接信息。

## 配置下游Allay服务器

下游Allay服务器同样需要启用网易客户端支持，请参考[启用网易支持](1-启用网易支持.md)。

此外，请确保每个下游Allay服务器监听的端口与WaterdogPE中配置的`address`端口一致。例如，若WaterdogPE中配置了`lobby`服务器地址为`127.0.0.1:19133`，则对应Allay服务器的端口应设置为`19133`。