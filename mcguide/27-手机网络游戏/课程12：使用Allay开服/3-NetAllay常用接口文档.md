---
front:
hard: 进阶
time: 30分钟
---

# NetAllay常用接口文档

本文整理了`NetAllay`项目中最常用、最适合插件开发直接使用的接口，方便您在Allay服务器中调用网易版独有能力，例如PyRpc通信与商城能力。

如果您尚未安装NetAllay，请先阅读[启用网易支持](1-启用网易支持.md)。

## 获取NetAllay实例

调用接口前，先拿到`NetAllay`插件实例：

```java
import org.allaymc.api.server.Server;
import org.allaymc.netallay.NetAllay;

// 方式一：直接获取静态实例
NetAllay netAllay = NetAllay.getInstance();

// 方式二：通过插件管理器获取
NetAllay netAllay = (NetAllay) Server.getInstance()
        .getPluginManager()
        .getPlugin("NetAllay");
```

## PyRpc事件监听

### 监听客户端发来的事件

最常用的入口是`listenForEvent`，用于监听网易客户端发送到服务端的PyRpc事件。

```java
netAllay.listenForEvent(
        "MyMod",
        "MySystemCS",
        "OnButtonClick",
        (player, data) -> {
            String buttonId = (String) data.get("buttonId");
            plugin.getPluginLogger().info("玩家点击了按钮: {}", buttonId);
        }
);
```

参数说明：

- `namespace`：命名空间，一般对应客户端Mod命名空间
- `systemName`：客户端系统名
- `eventName`：事件名
- `handler`：回调函数，参数为`Player`和`Map<String, Object>`

### 取消事件监听

如果您不再需要某个监听器，可以调用以下接口：

```java
boolean removed = netAllay.unlistenForEvent(namespace, systemName, eventName, handler);

netAllay.unlistenAllForEvent(namespace, systemName, eventName);
```

其中：

- `unlistenForEvent`：取消一个具体处理器
- `unlistenAllForEvent`：取消某个事件下的全部处理器

## 向客户端发送PyRpc事件

### 发送给单个玩家

```java
Map<String, Object> data = new LinkedHashMap<>();
data.put("message", "Hello from server!");
data.put("timestamp", System.currentTimeMillis());

boolean success = netAllay.notifyToClient(
        player,
        "MyMod",
        "MySystemSS",
        "ServerMessage",
        data
);
```

该方法返回`boolean`，表示是否发送成功。若玩家不是网易客户端，则会返回`false`。

### 立即发送

若您希望不经过缓冲立即下发，可以使用：

```java
boolean success = netAllay.notifyToClientImmediately(
        player,
        "MyMod",
        "MySystemSS",
        "ServerMessage",
        data
);
```

### 发送给多个玩家

```java
int successCount = netAllay.notifyToMultiClients(
        players,
        "MyMod",
        "MySystemSS",
        "Announcement",
        Map.of("content", "服务器公告")
);
```

返回值为成功发送的玩家数量。

### 发送给附近玩家

```java
var center = player.getControlledEntity().getLocation();

int count = netAllay.notifyToClientsNearby(
        null,
        center,
        50.0,
        "MyMod",
        "MySystemSS",
        "NearbyEvent",
        Map.of("type", "explosion")
);
```

参数中的`except`可用于排除某个玩家；若不需要排除，传`null`即可。

### 广播到世界、维度或全服

广播接口统一为`broadcastToAllClient`，但有三个重载版本：

```java
// 广播到指定世界
int worldCount = netAllay.broadcastToAllClient(
        null,
        world,
        "MyMod",
        "MySystemSS",
        "WorldEvent",
        data
);

// 广播到指定维度
int dimensionCount = netAllay.broadcastToAllClient(
        null,
        dimension,
        "MyMod",
        "MySystemSS",
        "DimensionEvent",
        data
);

// 广播到整个服务器
int serverCount = netAllay.broadcastToAllClient(
        null,
        "MyMod",
        "MySystemSS",
        "GlobalEvent",
        data
);
```

## 工具接口

### 判断玩家是否为网易客户端

```java
boolean netease = netAllay.isNetEasePlayer(player);
```

### 获取当前注册的事件数量

```java
int count = netAllay.getRegisteredEventCount();
```

## 特殊常量

### `LOCAL_PLAYER_ENTITY_ID`

```java
int entityId = NetAllay.LOCAL_PLAYER_ENTITY_ID; // -2
```

这个常量表示“接收消息的玩家自己”。常见用法是把实体ID传给客户端时，使用`-2`代表当前玩家自身实体。

```java
netAllay.notifyToClient(
        player,
        "MyMod",
        "MySystemSS",
        "Welcome",
        Map.of("entityId", NetAllay.LOCAL_PLAYER_ENTITY_ID)
);
```

注意：`-2`只能在单播接口`notifyToClient`或`notifyToClientImmediately`中使用，不要在多播或广播接口中使用。

## 事件数据支持的类型

`NetAllay`的事件数据类型为`Map<String, Object>`，其中值通常支持以下类型：

- `null`
- `Boolean`
- `Integer`、`Long`
- `Float`、`Double`
- `String`
- `byte[]`
- `Map<String, Object>`
- `List<?>`、`Iterable<?>`

一个常见示例：

```java
Map<String, Object> data = new LinkedHashMap<>();
data.put("title", "测试标题");
data.put("count", 1);
data.put("extra", Map.of("flag", true));
data.put("list", List.of("a", "b", "c"));
```

## 商城接口

`NetAllay`内置了商城管理器`ShopManager`，用于控制网易商城界面与监听商城事件。

### 获取商城管理器

```java
var shopManager = netAllay.getShopManager();
```

### 控制商城入口与界面

```java
// 是否启用自定义商城入口
netAllay.enableCustomShopEntry(true);

// 打开商城
netAllay.openShop(player);

// 关闭商城
netAllay.closeShop(player);

// 显示一行提示
netAllay.showHint(player, "购买成功");

// 显示两行提示
netAllay.showHint(player, "购买成功", "奖励已发放");
```

这些方法本质上是对`ShopManager`的便捷封装。

### 监听商城事件

商城常见事件定义在`ShopEvent`中：

- `ShopEvent.PLAYER_BUY_ITEM_SUCCESS`：玩家购买成功
- `ShopEvent.PLAYER_URGE_SHIP`：玩家催发货
- `ShopEvent.CLIENT_LOAD_ADDON_FINISH`：客户端加载Addon完成

注册示例：

```java
shopManager.listenForShopEvent(ShopEvent.PLAYER_BUY_ITEM_SUCCESS, (player, data) -> {
    plugin.getPluginLogger().info("玩家 {} 完成了一笔购买", player.getOriginName());
});

shopManager.listenForShopEvent(ShopEvent.PLAYER_URGE_SHIP, (player, data) -> {
    plugin.getPluginLogger().info("玩家 {} 请求催发货", player.getOriginName());
});
```

取消监听：

```java
boolean removed = shopManager.unlistenForShopEvent(ShopEvent.PLAYER_BUY_ITEM_SUCCESS, handler);
```

### 按玩家覆盖商城配置

如果您希望不同玩家拿到不同的商城参数，可以为玩家设置单独配置：

```java
ShopManager.PlayerShopConfig config = new ShopManager.PlayerShopConfig();
config.setGameId("123456");
config.setUseCustomShop(true);
config.setTestServer(false);
config.setCacheTime(1);
config.setUid(0L);
config.setPlatformUid("");

shopManager.setPlayerConfig(player, config);
```

获取和移除配置：

```java
ShopManager.PlayerShopConfig config = shopManager.getPlayerConfig(player);
shopManager.removePlayerConfig(player);
```

## 配置文件

NetAllay会在插件目录生成`config.json`。当前源码中的默认配置如下：

```json
{
  "shop": {
    "gameId": "",
    "gameKey": "",
    "testGameKey": "",
    "testServer": false,
    "useCustomShop": false,
    "cacheTime": 1,
    "shopServerUrl": "",
    "webServerUrl": ""
  }
}
```

其中商城相关字段含义如下：

- `gameId`：资源数字ID
- `gameKey`：正式服签名密钥
- `testGameKey`：测试服签名密钥
- `testServer`：是否使用测试服模式
- `useCustomShop`：是否隐藏默认商城按钮并改为自定义入口
- `cacheTime`：商城配置缓存时间，单位为秒
- `shopServerUrl`：自定义订单服务地址，可留空
- `webServerUrl`：自定义Web服务地址，可留空

如果您在运行中修改了配置，也可以调用以下接口：

```java
netAllay.reloadConfig();
netAllay.saveConfig();
```

## 完整示例

```java
import org.allaymc.api.player.Player;
import org.allaymc.netallay.NetAllay;
import org.allaymc.netallay.shop.ShopEvent;

import java.util.Map;

public final class DemoService {

    public void register() {
        NetAllay netAllay = NetAllay.getInstance();

        netAllay.listenForEvent("MyMod", "MySystemCS", "RequestData", (player, data) -> {
            netAllay.notifyToClient(
                    player,
                    "MyMod",
                    "MySystemSS",
                    "ResponseData",
                    Map.of(
                            "success", true,
                            "playerName", player.getOriginName()
                    )
            );
        });

        netAllay.getShopManager().listenForShopEvent(ShopEvent.PLAYER_BUY_ITEM_SUCCESS, (player, data) -> {
            netAllay.showHint(player, "购买成功", "请及时发货");
        });
    }

    public void welcome(Player player) {
        NetAllay.getInstance().notifyToClient(
                player,
                "MyMod",
                "MySystemSS",
                "Welcome",
                Map.of(
                        "entityId", NetAllay.LOCAL_PLAYER_ENTITY_ID,
                        "message", "欢迎来到服务器"
                )
        );
    }
}
```

## 注意事项

- `NetAllay`只会向网易客户端发送数据，非网易玩家会被自动跳过
- 广播、多播、附近广播时不要使用`NetAllay.LOCAL_PLAYER_ENTITY_ID`
- 插件禁用时，建议取消自己注册的事件监听
- 单次发送的数据不宜过大，避免发送过大的PyRpc包
