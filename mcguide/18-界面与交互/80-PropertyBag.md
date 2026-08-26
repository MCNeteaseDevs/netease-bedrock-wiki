# PropertyBag 简易教程

#### 作者：Dofes

本文介绍 Bedrock JSON UI 中 `PropertyBag` 的概念、机制和使用方法。

开发者在接触 JSON UI 时，通常首先关注 `controls`、`bindings`、`variables` 和 `ViewBinder` 等内容。但在构建复杂界面、改造原生 UI 或复用引擎内置 renderer 时，`PropertyBag` 起着关键作用。

`PropertyBag` 是 UI 控件在运行时维护的一份键值对表。它接收来自 JSON `property_bag` 字段的初始值、`bindXXX` 等绑定接口写入的数据，以及 Python 侧 `SetPropertyBag()` 接口的修改。许多引擎内置 UI 行为的本质，就是向控件的 `PropertyBag` 写入数据，再由另一段逻辑读取并执行。

# 概述

`PropertyBag` 与普通 JSON 变量的区别在于：普通变量偏向静态定义和表达式求值，而 `PropertyBag` 是一份在运行时持续变化的上下文。它包含以下来源的数据：

- 控件的初始状态（来自 JSON `property_bag`）
- 绑定系统写入的结果
- Python 脚本写入的值
- 输入事件、集合上下文、renderer 私有协议等运行时信息

# 什么是 PropertyBag

从使用层面来看，`PropertyBag` 是每个控件在运行时维护的一份键值对映射。

在 JSON 中，你可以这样声明它：

```json
"property_bag": {
  "#visible": true,
  "#my_mod.some_flag": false
}
```

在 Python 中，你可以这样读取和修改它：

```python
ctrl = self.GetBaseUIControl("/panel/example")
bag = ctrl.GetPropertyBag()
ctrl.SetPropertyBag({"#my_mod.some_flag": True})
```

需要注意的是，`PropertyBag` 不仅仅是供开发者自定义字段的容器。许多引擎内置控件、组件和 renderer 会主动读取其中的特定字段，将其作为输入协议。写入这些字段后，界面会相应发生变化。

# PropertyBag 的生命周期

理解 `PropertyBag` 的关键在于理解它的生命周期。

## 控件创建时

如果某个控件在 JSON 中声明了 `property_bag`，那么这份数据会作为控件的初始 `PropertyBag` 被创建出来。

如果 JSON 中没有声明，引擎仍可能在后续读写时为控件自动创建一份空的 `PropertyBag`。

因此，`SetPropertyBag()` 不要求事先在 JSON 中声明空的 `property_bag`。对许多控件而言，首次读写时引擎会自动创建对应的 `PropertyBag`。

## 运行过程中

一个控件的 `PropertyBag` 并不是只在创建时初始化一次。它在整个控件生命周期中都可能被不断修改。常见来源包括：

- JSON 中的初始 `property_bag`
- `bindings` 与 `ViewBinder.bindXXX`
- `view` 类型绑定带来的属性传播
- Python 侧的 `SetPropertyBag()`
- 输入事件、集合信息、控件状态更新
- 某些 renderer 或组件在运行时写回的状态

也就是说，同一个控件的 `PropertyBag` 通常由多方共同维护。

## 什么时候真正生效

`PropertyBag` 本身只是数据容器。往里面写值，并不等于界面一定会发生变化。

一个字段要生效，至少要满足下面之一：

- 有 `bindings` 把它映射到了某个可见属性
- 有内置 component 读取它
- 有内置 renderer 读取它
- 有某段 UI 逻辑把它当成协议字段来消费

如果没有消费者，该值仅存储于 `PropertyBag` 中，不会产生任何可见效果。

## 控件销毁时

`PropertyBag` 归属于控件本身，因此它的生命周期通常与控件一致。

这意味着：

- 关闭一个 screen 后，screen 上控件的 `PropertyBag` 也会随之销毁
- 通过 factory 动态创建的子控件被销毁后，它们的 `PropertyBag` 也会消失
- 不要将 `PropertyBag` 作为持久化存储

重新打开界面时，即使路径完全相同，通常也会创建全新的控件树，对应的 `PropertyBag` 也随之重建。

## `GetPropertyBag()` 得到的是什么

Python 侧 `GetPropertyBag()` 返回的是当前 `PropertyBag` 的快照，可用于观察当前状态和调试字段。该返回值并非引用，故修改该返回值不会自动写回控件——修改界面需要通过 `SetPropertyBag()` 进行。

# SetPropertyBag 与 bindXXX

`SetPropertyBag()` 和 `bindXXX` 最终都会写入同一份 `PropertyBag`，区别在于驱动模型不同。

## 相同点

1. 两者最终都会将值写入控件的 `PropertyBag`。

2. 两者可以驱动同一批下游消费者。例如 `#item_id_aux` 既可以来自 `bindXXX`，也可以来自 `SetPropertyBag()`，消费它的都是同一个物品 renderer。

3. 两者都可以参与原生 UI 协议。引擎内置字段不关心值的来源是绑定还是脚本，只要值出现在正确的控件 `PropertyBag` 中即可生效。

## 区别

两者的关键区别在于驱动模型不同。

`bindXXX` 偏向声明式模型：开发者声明”当某个绑定被请求时应返回什么值”，由引擎在合适的时机调用绑定回调并将结果写入 `PropertyBag`。适合持续变化的数据、集合项数据以及需要与 JSON `bindings` 长期协作的场景。

`SetPropertyBag()` 偏向命令式模型：在某个时刻明确向控件写入一批字段，等待下游消费。适合临时修改、一次性触发效果或向 renderer 传入协议字段。

从覆盖关系来看，`SetPropertyBag()` 写入的值不会锁定字段。如果同一个 key 同时被 `bindXXX` 持续维护，后续绑定重新执行时可能覆盖手动写入的值。

因此，选择哪一种方式，需要根据 renderer 的实现和数据的生命周期来判断。

## 一个简单的判断方法

| 场景 | 更适合的方式 |
| --- | --- |
| 某个文本、布尔值、数字需要长期和 Python 状态同步 | `bindXXX` |
| 某个集合项需要按 index 动态返回数据 | `binding_collection` / 集合绑定 |
| 想立刻改一个控件当前的运行时状态 | `SetPropertyBag()` |
| 想临时触发一次动画、一次 tooltip、一次 renderer 行为 | `SetPropertyBag()` |
| 想让 JSON 的 `bindings` 体系长期托管这个值 | `bindXXX` |

# SetPropertyBag 的特点

`SetPropertyBag()` 在 Python 中的行为是逐字段写入，而非替换整个 bag。传入的每个字段会单独 `set` 到现有 PropertyBag 中，未提及的字段不受影响。

```python
ctrl.SetPropertyBag({
    "#hover_text": "hello",
    "#visible": True
})
```

上述调用只会修改 `#hover_text` 和 `#visible`，bag 中其他字段保持不变。

这种灵活性也带来两个常见问题：

一是与绑定的冲突。如果某个字段正被绑定系统持续维护，手动写入的值可能在下一帧就被覆盖。

二是字段缺少消费者。写入的键名虽然合理，但如果没有任何内置逻辑读取该字段，不会产生任何效果。

另外，Python 侧 API 没有暴露删除字段的能力。`SetPropertyBag()` 只能写入或覆盖字段，无法清除已有字段。字段会一直存在于 bag 中，直到控件销毁。因此应避免向 bag 写入大量一次性使用的不同 key，否则 bag 会持续膨胀。

# PropertyBag 与 JSON `property_bag`

开发中最容易混淆的是这三者：

- JSON 中的 `property_bag`
- `bindXXX`
- `SetPropertyBag()`

三者虽然都会写入 `PropertyBag`，但职责不同：

- JSON `property_bag`：初始值。适合定义控件创建时就应生效的状态，例如控件是否属于某个 renderer 组，或某个开关的默认值。
- `bindXXX`：持续供数。适合由 Python 状态源长期维护的值。
- `SetPropertyBag()`：运行时写入。适合不需要搭建绑定链路但需要在某一时刻生效的操作。

# 使用 PropertyBag 时的几个常见误区

## 误区一：写入即生效

`PropertyBag` 只是输入层。字段是否生效取决于有无后续消费者。没有人读取，该值仅存储于 bag 中。

## 误区二：SetPropertyBag 可以替代所有绑定

`SetPropertyBag()` 并不适合替代所有绑定。如果一个值需要持续更新、需要和集合索引联动、需要稳定地被引擎反复读取，绑定体系仍然是更合适的做法。

## 误区三：GetPropertyBag 返回的是引用

`GetPropertyBag()` 返回的是当前状态的快照，而非引用。修改返回的 dict 不会影响 UI。

## 误区四：PropertyBag 可以作为持久化数据层

`PropertyBag` 的生命周期与控件一致。它适用于运行时状态，不适用于长期保存。

## 误区五：复杂类型可以直接塞进去

`SetPropertyBag()` 在 Python 侧能直接处理的仅限基础标量类型。复杂结构需要按引擎约定转换为特定格式的字符串。

例如 `flying_item_user_data` 当前接收的是 `CompoundTag` 二进制序列化后 base64 编码的字符串，这属于引擎约定而非接口固有限制。

# renderer 私有协议字段

所谓”renderer 私有协议字段”，指的是引擎内置 renderer 会主动读取的一组字段。这些字段通常不会在常规 UI 文档中集中列出，但在很多场景下具有实际用途。

需要特别说明的是：这一部分本质上属于非官方协议。它们往往来自现有原生界面的实际行为和长期测试经验，因此虽然非常实用，但仍然可能随着版本变化而调整。公开使用时，最好把它们当作“已知可用的运行时接口”，而不是稳定不变的正式 API。

本节优先整理适合在 Python 侧直接通过 `SetPropertyBag()` 驱动的字段。

这里需要先说明一个前提：Python 侧 `SetPropertyBag()` 当前只支持四类值：

- `bool`
- `int`
- `float`
- `string`

除此以外，Python 侧 `SetPropertyBag()` 不支持直接写入数组、对象或其他复杂结构。因此下面的表格，会优先整理这类“可以直接从 Python 写入”的字段。

文中提到的“默认值”，指的是字段缺失时的常见行为。

## `hover_text_renderer`

这是最容易复用的一类私有协议。

| 字段          | 含义                 | 期望类型 | 默认值 | 备注 |
| ------------- | -------------------- | -------- | ------ | ---- |
| `#hover_text` | 控制悬浮提示文本内容 | `string` | `""`   |      |

如果目标控件本身由 hover text 类 renderer 驱动，那么运行时写入 `#hover_text`，通常就能直接改掉显示内容。

## `inventory_item_renderer`

这是最常用的一类私有协议字段。它不仅能显示普通物品，还能复用原版物品渲染逻辑去显示附魔、旗帜、盾牌、纹饰等效果。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `#item_id_aux` | 物品渲染的核心字段，决定显示哪个物品 | `int` | `-1` |  |
| `#charged_item` | 某些充能类物品的附加渲染信息 | `int` | `0` |  |
| `#item_custom_color` | 自定义颜色 | `int` | `0xFFFFFFFF` |  |
| `#item_pickup_time` | 拾取时间相关输入 | `int` | `0` |  |
| `#show_item_pickup` | 是否显示拾取相关效果 | `bool` | `true` |  |
| `#fade_in_icon_time_seconds` | 图标淡入相关时间 | `float` | `0.0` |  |
| `#item_lodestone_tracking_handle` | 指南针/磁石追踪句柄 | `int` | `0` |  |
| `#armor_trim_material` | 盔甲纹饰材质 | `string` | `""` |  |
| `#banner_patterns` | 旗帜图案数据 | `string` | `""` |  |
| `#banner_colors` | 旗帜颜色数据 | `string` | `""` |  |
| `#shield_base_color` | 盾牌底色 | `int` | `-1` |  |
| `#shield_is_active` | 盾牌激活状态 | `bool` | 建议显式传入 |  |
| `#decorated_pot_sherds` | 饰纹陶罐碎片数据 | `string` | `""` |  |
| `#serialized_actor_pose` | 某些物品姿态信息 | `int` | `0` |  |
| `#should_show_bundle_open_back` | bundle 打开状态背面效果 | `bool` | `false` |  |
| `#should_show_bundle_open_front` | bundle 打开状态正面效果 | `bool` | `false` |  |
| `#item_mod_value` | 自定义物品扩展值 | `int` | `-1` |  |
| `#item_mod_extend_value` | 自定义物品扩展字符串 | `string` | `""` |  |
| `#item_mod_id` | 自定义物品模组标识 | `string` | `""` |  |
| `#item_mod_item_id` | 自定义物品条目标识 | `string` | `""` |  |
| `#item_micro_item_value` | 微型物品值 | `string` | `""` |  |

这套协议的核心字段是 `#item_id_aux`。多数场景下，通过 `GetItemBasicInfo(...)[\"id_aux\"]` 获取该值即可驱动物品图标。

## `equipment_preview_renderer`

`equipment_preview_renderer` 用于在装备预览控件中展示装备，而非 2D 图标槽中的物品图标。

| 字段                   | 含义             | 期望类型 | 默认值       | 备注 |
| ---------------------- | ---------------- | -------- | ------------ | ---- |
| `#item_id_aux`         | 要预览的装备物品 | `int`    | `0`          |      |
| `#item_custom_color`   | 自定义颜色       | `int`    | `0xFFFFFFFF` |      |
| `#armor_trim_pattern`  | 纹饰图案         | `string` | `""`         |      |
| `#armor_trim_material` | 纹饰材质         | `string` | `""`         |      |

## `flying_item_renderer`

`flying_item_renderer` 是完全由私有协议驱动的 renderer。

`flying_item_count` 决定本次新增的飞行物品数量。renderer 按 `0` 到 `count-1` 的序号读取每个飞行物品的字段，例如 `flying_item_id_aux0`、`flying_item_id_aux1`、`flying_item_id_aux2`。处理完毕后 `flying_item_count` 会被重置为 `0`。

这种 `count` + 序号的设计具有批处理语义：一次 `SetPropertyBag()` 调用同时提交多个飞行物品，renderer 在同一帧内统一创建，确保各物品的动画起始时间一致。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `flying_item_count` | 本次要新增的飞行物品数量 | `int` | `0` |  |
| `flying_item_id_auxN` | 第 `N` 个飞行物品的 `id_aux` | `int` | `0` |  |
| `flying_item_origin_position_xN` | 起点 x | `float` | `0.0` |  |
| `flying_item_origin_position_yN` | 起点 y | `float` | `0.0` |  |
| `flying_item_origin_scaleN` | 起点缩放 | `float` | `1.0` |  |
| `flying_item_destination_position_xN` | 终点 x | `float` | `0.0` |  |
| `flying_item_destination_position_yN` | 终点 y | `float` | `0.0` |  |
| `flying_item_destination_scaleN` | 终点缩放 | `float` | `1.0` |  |
| `flying_item_user_dataN` | 物品附加数据 | `string` | `""` | 当前要求 `CompoundTag` 二进制序列化后 base64 编码 |
| `flying_item_mod_valueN` | 自定义物品扩展值 | `int` | `-1` |  |
| `flying_item_mod_extend_valueN` | 自定义物品扩展字符串 | `string` | `""` |  |

该 renderer 的特点是：只要字段齐全即可工作，不需要 `#text`、`#visible` 等常规绑定字段。

## `panorama_renderer`

该 renderer 用于运行时切换背景、拖动和横向偏移。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `#texture` | 全景贴图路径 | `string` | `""` |  |
| `#texture_file_system` | 贴图来源类型 | `string` | `""` |  |
| `#uv_offset_x` | 横向偏移量 | `float` | `0.0` |  |
| `#gesture_mouse_delta_x` | 横向拖拽输入 | `float` | `0.0` | 消费后清零 |
| `#scroll_button_release` | 某些滚动释放状态 | `bool` | `false` |  |

其中 `#uv_offset_x` 和 `#gesture_mouse_delta_x` 都更接近“瞬时输入”，适合用来触发一次位移，而不是长期保存。

## `animated_gif_renderer`

该 renderer 用于展示 GIF 动图。

| 字段         | 含义         | 期望类型 | 默认值  | 备注 |
| ------------ | ------------ | -------- | ------- | ---- |
| `#gif_path`  | GIF 资源路径 | `string` | `""`    |      |
| `#pause_gif` | 是否暂停播放 | `bool`   | `false` |      |

## `progress_bar_renderer`

该 renderer 用于复用原版风格的进度条，无需手动计算宽度裁剪。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `#progress_bar_visible` | 是否显示进度条 | `bool` | `false` |  |
| `#touch_progress_bar_visible` | 触屏模式下是否显示 | `bool` | `false` |  |
| `#progress_bar_total_amount` | 总量 | `float` | `0.0` |  |
| `#progress_bar_current_amount` | 当前量 | `float` | `0.0` |  |
| `is_storage_bar` | 是否按存储条逻辑渲染 | `bool` | `false` |  |
| `is_durability` | 是否按耐久条逻辑渲染 | `bool` | `false` |  |
| `drop_shadow` | 是否绘制阴影 | `bool` | `false` |  |
| `round_value` | 是否对值取整 | `bool` | `false` |  |

## `enchanting_book_renderer`

| 字段    | 含义                 | 期望类型 | 默认值  | 备注 |
| ------- | -------------------- | -------- | ------- | ---- |
| `#open` | 控制附魔书打开或闭合 | `bool`   | `false` |      |

## `qr_code_renderer`

| 字段               | 含义             | 期望类型 | 默认值 | 备注 |
| ------------------ | ---------------- | -------- | ------ | ---- |
| `#qr_code_content` | 二维码内容字符串 | `string` | `""`   |      |

## `bohr_model_renderer`

| 字段             | 含义               | 期望类型 | 默认值  | 备注           |
| ---------------- | ------------------ | -------- | ------- | -------------- |
| `electron_count` | 电子数量           | `int`    | `0`     |                |
| `proton_count`   | 质子数量           | `int`    | `0`     |                |
| `neutron_count`  | 中子数量           | `int`    | `0`     |                |
| `init_model`     | 是否重新初始化模型 | `bool`   | `false` | 消费后自动归位 |

## `paper_doll_renderer`

渲染玩家皮肤的 3D 纸偶模型，支持多种旋转模式、皮肤切换、动画播放和头部追踪。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `#skin_rotation` | 是否启用自动旋转 | `bool` | `false` |  |
| `#custom_rot_y` | 指定 Y 轴旋转角度（度） | `float` | `0.0` |  |
| `#set_target_rotation` | 触发平滑旋转到指定 Y 轴角度，消费后清除 | `float` | 当前值 |  |
| `#gesture_delta_source` | 手势输入源类型 | `int` | `0` |  |
| `#gesture_mouse_delta_x` | 横向拖拽输入，消费后归零 | `float` | `0.0` |  |
| `#pack_id` | 皮肤包 ID | `int` | `0` |  |
| `#skin_idx` | 皮肤在包内的索引 | `int` | `-1` |  |
| `#player_uuid` | 按 UUID 查找玩家皮肤 | `string` | `""` |  |
| `#force_skin_update` | 强制重新加载皮肤 | `bool` | `false` |  |
| `#animation_to_play` | 一次性播放的动画名称，消费后清除 | `string` | `""` |  |
| `#animation_to_play_when_done` | 一次性动画结束后的过渡动画 | `string` | `"default"` |  |
| `#sound_to_play` | 动画触发时播放的音效名，消费后清除 | `string` | `""` |  |
| `#disable_head_follow_mouse` | 禁用头部跟随鼠标 | `bool` | `false` |  |
| `#force_dirty_animation` | 强制标记动画为脏 | `bool` | `false` |  |

## `actor_portrait_renderer`

渲染实体的 3D 肖像，支持缩放、旋转、平移、自动旋转和皮肤覆盖。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `actor_id` | 要渲染的实体唯一 ID | `int` | `0` |  |
| `scale_x` | X 轴缩放 | `float` | `1.0` |  |
| `scale_y` | Y 轴缩放 | `float` | `1.0` |  |
| `scale_z` | Z 轴缩放 | `float` | `1.0` |  |
| `rotate_x` | X 轴旋转（度） | `float` | `0.0` |  |
| `rotate_y` | Y 轴旋转（度） | `float` | `0.0` |  |
| `rotate_z` | Z 轴旋转（度） | `float` | `0.0` |  |
| `translate_x` | X 轴平移 | `float` | `0.0` |  |
| `translate_y` | Y 轴平移 | `float` | `0.0` |  |
| `translate_z` | Z 轴平移 | `float` | `0.0` |  |
| `use_live_animation` | 使用实体实时动画 | `bool` | `false` |  |
| `rotate_mode` | 旋转模式，`"auto"` 为自动旋转 | `string` | `""` |  |
| `skin` | 皮肤覆盖 JSON 字符串 | `string` | `""` |  |

## `name_tag_renderer`

渲染带背景的玩家名牌。

| 字段          | 含义         | 期望类型 | 默认值 | 备注 |
| ------------- | ------------ | -------- | ------ | ---- |
| `#playername` | 玩家名称文本 | `string` | `""`   |      |
| `#x_padding`  | 水平内边距   | `int`    | `0`    |      |
| `#y_padding`  | 垂直内边距   | `int`    | `0`    |      |

## `live_player_renderer`

在 UI 中渲染本地玩家的 3D 实时模型（如背包界面）。

| 字段              | 含义                       | 期望类型 | 默认值  | 备注 |
| ----------------- | -------------------------- | -------- | ------- | ---- |
| `#look_at_cursor` | 头部和身体是否跟随光标旋转 | `bool`   | `false` |      |

## `live_horse_renderer`

在 UI 中渲染马的 3D 实时模型（马匹背包界面）。

| 字段         | 含义                          | 期望类型 | 默认值 | 备注 |
| ------------ | ----------------------------- | -------- | ------ | ---- |
| `#entity_id` | 马实体的唯一 ID（字符串形式） | `string` | `""`   |      |

## `netease_paper_doll_renderer`

中国版扩展的纸偶渲染器，支持三种渲染模式：实体、骨骼模型、方块几何体。支持三轴自由手势旋转。

| 字段 | 含义 | 期望类型 | 默认值 | 备注 |
| --- | --- | --- | --- | --- |
| `#gesture_delta_source` | 手势输入源类型 | `int` | `0` |  |
| `#gesture_mouse_delta_x` | 横向拖拽输入，消费后归零 | `float` | `0.0` |  |
| `#gesture_mouse_delta_y` | 纵向拖拽输入（仅自由手势模式），消费后归零 | `float` | `0.0` |  |
| `#custom_rot_y` | 指定 Y 轴旋转角度（度） | `float` | `0.0` |  |

# 与 renderer 私有协议字段打交道时的建议

使用 renderer 私有协议字段时，建议遵循以下原则：

1. 优先使用语义明确、行为稳定、已有原生界面使用的字段，例如 `#hover_text`、`#item_id_aux`。这类字段更容易理解和调试。

2. 将这类字段视为引擎协议而非普通变量。协议字段的值格式由引擎约定，可能随版本变化。例如 `flying_item_user_data` 当前要求 `CompoundTag` 二进制序列化后 base64 编码的字符串。

3. 按最小闭环进行测试。先确认控件确实由目标 renderer 驱动，再写入最少的一组字段并验证效果，最后逐步添加更多字段。

# 使用 PropertyBag 时的几个坑

## 1. 没有消费者

字段名正确不代表界面一定有变化，前提是必须有逻辑读取该字段。

## 2. 被绑定系统覆盖

如果同一个 key 正被 `bindXXX` 持续维护，`SetPropertyBag()` 写入的值可能在后续更新中被覆盖。

## 3. 类型限制

直接写入时，当前只支持：

- `bool`
- `int`
- `float`
- `string`

除此以外的类型，Python 侧 `SetPropertyBag()` 不能直接写入。

对于语义上承载复杂数据但接口层面接收字符串的字段，其值的格式由引擎约定决定，可能随版本变化。例如 `flying_item_user_dataN` 当前要求 `CompoundTag` 二进制序列化后 base64 编码的字符串，但这只是引擎当前的约定，并非接口固有限制。

## 4. 布局相关字段的副作用

某些 `PropertyBag` 字段会间接影响布局、焦点或集合逻辑。写入不合适的控件可能导致布局异常、点击逻辑错乱，甚至触发难以定位的断言。

## 5. 生命周期误判

`PropertyBag` 是控件的局部运行时状态，而非 screen 级别的全局状态。控件销毁后，其 `PropertyBag` 也随之销毁。

## 6. 字段无法删除

Python 侧 API 只能写入或覆盖字段，无法清除已有字段。字段一旦写入就会一直存在于 bag 中，直到控件销毁。

# 总结

`PropertyBag` 是 Bedrock JSON UI 运行时控制面的重要组成部分。它连接了 JSON `property_bag`、`bindings`、`ViewBinder`、运行时脚本以及各类内置 renderer。

理解 `PropertyBag` 后，UI 开发问题可以按以下思路拆解：

- 这个效果由谁消费数据？
- 数据最终写入哪个控件？
- 该值适合用绑定持续维护，还是用 `SetPropertyBag()` 临时触发？

`bindXXX` 提供声明式数据通道，`SetPropertyBag()` 提供命令式运行时写入接口。两者并非互斥关系，而是各自适用于不同场景。
