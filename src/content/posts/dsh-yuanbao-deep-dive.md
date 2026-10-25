---
title: 手搓 dsh-yuanbao：从「群里 @ 它不回」到「引用消息全打通」
published: 2026-10-25T21:30:00
description: '从「群里 @ 它一次都不回」开始，记录给 DSH 写元宝通道插件的完整过程，以及官方原版的几个真实问题。'
tags: [DSH, 插件开发, 逆向, 踩坑记录]
draft: false
lang: 'zh_CN'
---

## 起因

我在用 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（下称 DSH）做日常开发。它是个插件化很彻底的 Agent 框架——「一切皆插件」，连通道接入都是插件。

元宝官方给 DSH 写了一个通道插件 `dsh-yuanbao`，功能是：把元宝 IM 的消息接进 DSH，让 Agent 能在群里聊天、回消息。

装上之后，私聊是通的。但一进群就出问题：

**在群里 @ 它，它不回。**

不是偶尔不回，是**一次都不回**。消息发出去，界面上什么反应都没有，就像那个 @ 从来没发生过。

这就是一切的起点。

---

## 官方原版的几个真实问题

先说清楚：**这不是黑官方**。`dsh-yuanbao` 是 MIT 协议的，能用，基础功能也没问题。但它的定位决定了它没被认真打磨——

| 指标 | `dsh-yuanbao` | `yuanbao-openclaw-plugin` |
|---|---|---|
| 代码行数 | 5,747 | 16,458 |
| 源码文件 | 31 | 115 |
| **测试文件** | **0** | **72** |
| 最后更新 | 2026-08-15（创建即停更） | 2026-10-20（活跃） |

**零测试、创建即停更、两个月无维护。** 所以下面这些问题，本质上不是「写错了」，而是「没人测过」。

### 问题一：@提及判定失效，所有群消息都不回

群聊里判断「有没有 @ 我」时，原版是这么写的：

```typescript
if (customContent?.user_id === botId) { ... }
```

问题是：如果 `botId` 拿不到（比如 sign-token 还没换回来），这个比较永远是 `false`，插件就认为「没人 @ 我」，于是**所有群消息都被丢弃**。

而且它不区分两种情况：
- 确实没人 @ 我
- 我根本没有身份可以比对

**这就是我踩到的第一个坑。** 表面上「插件装好了、也连上了」，但群里怎么 @ 都没反应——因为它在更早的一步就把消息丢掉了，连日志都只是轻描淡写一句「group message without mention」。

**修法**：优先用配置的 `botId`，否则回退到 sign-token 解析出的 id；两种情况分开打日志。

### 问题二：群消息发不出去（`bot_id is required`）

解决完 @提及，群消息终于能进来了。但接着发现：**回复发不出去。**

平台返回「参数无效：bot_id is required」。

翻 `bridge.ts` 才发现，发群消息时会单独构造一个 account 对象：

```typescript
// 原版写法（有问题）
const account = gatewayAccountFromConfig(this.config);
```

这个新对象上 `botId` 是 `undefined`。而元宝平台校验发送者身份时，要求 `fromAccount` 必须有值。

**修法**：复用 gateway 已经解析好的 account 对象。

```typescript
// Reuse the gateway's account: sign-token resolution stored the bot id on
// that exact object. Building a second account here would leave `botId`
// undefined and make every group send fail the `fromAccount` check.
const account = gateway.account ?? gatewayAccountFromConfig(this.config);
```

### 问题三：发送失败还谎报成功

原版的发送逻辑，无论底层成没成功，都往上报「已发送」。结果就是：Agent 以为自己发出去了，实际上什么都没到用户手上。

这是最难排查的一类 bug——**表面上一切正常**。

**修法**：传播真实传输结果。

```typescript
// Propagate the transport outcome: reporting success unconditionally is
// what made the tool claim "sent" while nothing left the device.
```

### 问题四：附件被静默丢弃

原版的出站逻辑只处理纯文本。带图片/文件的消息会被过滤掉，附件直接消失。

**修法**：把文本和媒体分开收集，任何组合都能正确发送。

---

## 接着要解决的问题：引用消息

上面那些是「让它能正常在群里说话」的基础修复。

但真正让我下决心 fork 自己维护的，是另一个更麻烦的东西——**引用消息。**

### 元宝的消息结构限制

元宝里有个硬性设计：

```
图片消息 ≠ 文字消息
@bot 只能是文字消息
```

所以用户想让我看图时，实际做的是：

```
先发一张图（这条消息 @ 不到 bot）
再引用那张图 + @bot + 一句文字
```

**结果我收到的是「文字消息」，引用信息全丢了** —— 屏幕上只剩一个 `[image]`，图片内容什么都看不到。

### 引用信息藏在哪

翻元宝的协议文档，发现引用不在 `msg_body`（消息体元素）里，而在**消息级**的 `cloud_custom_data`：

```json
{
  "quote": {
    "desc": "被引用的文字内容",
    "type": 2,
    "id": "被引用消息的 msgId",
    "sender_nickname": "某人"
  }
}
```

`type` 的含义：

| 值 | 类型 |
|---|---|
| 1 | 文字 |
| 2 | 图片 |
| 3 | 文件 |
| 4 | 视频 |
| 5 | 语音 |

**关键点：媒体类引用的 `desc` 是空的。** 它只给你一个 `id`，剩下的自己去查。

### 实现

新建了两个模块。

**`src/quote.ts`** —— 解析引用：

```typescript
export function parseQuote(cloudCustomData: string | undefined): QuoteInfo | undefined
export function resolveQuoteMedia(quote: QuoteInfo, chatKey: string): InboundMedia | undefined
export function formatQuoteContext(quote: QuoteInfo, media?: InboundMedia): string
```

**`src/chat-history.ts`** —— 媒体历史缓存：

```typescript
// 收到媒体消息 → 记下 { messageId, medias[] }
// 收到引用消息 → 按 quote.id 回查 → 拿到原始 URL
export function rememberMedia(chatKey, entry)
export function findMediaByMessageId(chatKey, messageId)
```

**为什么要独立于消息历史**：群聊里 @bot 之后消息历史会被清理，但引用随时可能指向更早的图片。媒体历史单独存，生命周期更长（每会话 50 条 LRU）。

---

## 踩坑记录：八轮才打通

这部分是最真实的。**从「能识别引用」到「真正拿到图片」，我改了八轮。**

### 第 1 轮：引用完全没解析

**现象**：收到消息，引用信息全无。

**原因**：`cloud_custom_data` 这个字段虽然被协议层解码了，但业务代码完全没读它。

**修法**：新增 `quote.ts`。

### 第 2 轮：引用识别了，但媒体查不到

**现象**：日志显示 `mediaResolved=no`。

**原因**：没有媒体历史缓存，按 `id` 查不到。

**修法**：新增 `chat-history.ts`。

### 第 3 轮：记录时机错了

**现象**：缓存建好了，还是查不到。

**原因**：我把 `rememberMedia()` 放在了「@bot 检查」**之后**：

```
收到图片消息（没 @bot）
    ↓
requireMention 检查 → 没 @ → return   ← 在这里就退出了
    ↓
rememberMedia()  ← 永远执行不到
```

**修法**：把记录媒体**提前到 @bot 检查之前**。消息本身可以被规则丢弃，但它携带的媒体要先存档。

### 第 4 轮：文本显示还是 `[image]`

**现象**：日志里 `mediaResolved=yes`，但给 Agent 的上下文文本还是 `[image]`。

**原因**：`formatQuoteContext()` 只检查 `quote.desc`。媒体类引用的 `desc` 本来就是空的，所以它永远走 `else` 分支输出 `[image]`，完全不知道媒体已经解析出来了。

**修法**：给它加 `media` 参数。

```typescript
if (!desc) {
  if (media) desc = media.placeholder;   // 有实体 → 真实文件名
  else desc = `[${label}]`;              // 没实体 → 占位
}
```

### 第 5 轮：附件 `extracted=0`

**现象**：文件名显示对了，但图片没作为附件发出去。

**日志**：
```
inbound body shape: [{"type":"TIMCustomElem", ...}]
buildMediaBlocks: extracted=0; bodyLen=1
```

**原因**：引用消息的 `msg_body` 里只有一个 `TIMCustomElem`，没有图片元素。而我为了让引用媒体也走 `buildMediaBlocks`，构造了一个伪元素：

```typescript
{
  msg_type: 'TIMImageElem',
  msg_content: { url: quoteMedia.url, file_name: quoteMedia.name }
}
```

但 `extractMedias` 对图片**只从 `msg_content.image_info_array` 里取 URL**，根本不读 `msg_content.url`：

```typescript
if (elem.msg_type === 'TIMImageElem') {
  const info = content.image_info_array ?? [];
  const chosen = info[1] ?? info[0];
  const url = chosen?.url ?? '';    // ← 只认这里
  if (!url) continue;               // ← 伪元素永远走这条
}
```

**修法**：引用媒体本来就有现成的 `url` 和 `name`，不用再绕一圈。

```typescript
// 直接下载 + 存附件库
const downloaded = await downloadMedia(quoteMedia, logger);
const ref = await attachments.saveImage({ data, mediaType, name });
```

### 第 6 轮：下载返回 401

**现象**：
```
media download failed: ...jpeg; status=401
```

**原因**：入站媒体给的地址是 `https://hunyuan.tencent.com/api/resource/download?resourceId=xxx`，**这个地址需要鉴权头，匿名访问一律 401。**

**修法**（参考 openclaw 插件的做法）：

```
① 从 url 抠出 resourceId
② 带鉴权头调 GET /api/resource/v1/download?resourceId=xxx
③ 从返回里拿到真正的 COS 地址
④ 再下载那个地址
```

### 第 7 轮：返回结构不对

**现象**：
```
resolveDownloadUrl: exchange returned no url; keys=
                                              ↑ data 是空对象
```

**原因**：我假设返回是 `{ data: { url } }`，实际是根层级的 `{ realUrl: "..." }`。

**修法**：先打印完整响应看结构，再兼容多种字段名。

### 第 8 轮：终于通了

**日志**：
```
resolveDownloadUrl: raw response={"realUrl":"https://ybim01.tencentyuanbao.com/...jpeg?q-signature=..."}
inbound quoted media url resolved: name=..._811_720.jpeg
inbound quoted media attached as image: name=..._811_720.jpeg; bytes=288766
```

**288,766 字节的图片，成功作为 image 附件发给了 Agent。**

---

## 额外加的：设置面板

原版没有可视化配置，只能手写 `cordis.patch.yml` 或者设环境变量。我加了两个功能。

### 会话路由

元宝消息进哪个 DSH 会话，支持两种模式：

| 模式 | 行为 |
|---|---|
| `per-peer`（默认） | 原版行为：每个群聊/私聊各开一个独立会话 |
| `pinned` | 所有消息都路由到指定的那一个会话 |

`pinned` 模式下会话列表从 DSH 实时读取，不用手填 UUID。留空自动回退到 `per-peer`。

### 凭据配置

AppKey / AppSecret / Token 可以在面板里填，不用改配置文件。

**设计上做了几点：**

- **密钥不回显明文**：接口只返回掩码（`tes******123`）和「有没有设置」的标记
- **空串 = 不修改**：避免用户只想改会话路由时误删密钥
- **凭据改动需要重启**：WebSocket 连接是用旧凭据建立的，改了要重新握手

### 持久化

设置存到 `~/.dsh/.yuanbao-settings.json`，重启不丢。

**为什么不写 `cordis.patch.yml`**：那是静态配置，改一次要手写 YAML 再重启，体验太差。运行时设置放独立文件，互不干扰。

---

## 最终成果

```
22 files changed, 3558 insertions(+), 114 deletions(-)
```

### 新增模块

| 文件 | 行数 | 作用 |
|---|---|---|
| `src/tools.ts` | 407 | 5 个 Agent 工具 |
| `src/outbound-media.ts` | 363 | 出站媒体（COS 上传） |
| `client.js` | 327 | 设置面板 |
| `src/media.ts` | 203 | 入站媒体 |
| `src/quote.ts` | 148 | 引用解析 |
| `src/stickers.ts` | 131 | 表情包 |
| `src/chat-history.ts` | 94 | 媒体历史缓存 |
| `src/services.ts` | 59 | 服务读取封装 |

### Agent 工具

| 工具名 | 用途 |
|---|---|
| `yuanbao_send_media` | 发送图片或文件 |
| `yuanbao_send_sticker` | 发送表情包 |
| `yuanbao_search_stickers` | 搜索表情 |
| `yuanbao_send_message` | 发送纯文本 |
| `yuanbao_list_members` | 列出群成员 |

**意义**：原版只能「被动回复」，现在 Agent 可以主动发消息、发图、发表情。

### 已发布

```
@anxi78/dsh-yuanbao@0.2.1
https://www.npmjs.com/package/@anxi78/dsh-yuanbao
```

```bash
dsh plugin --profile web add @anxi78/dsh-yuanbao
```

---

## 一点感想

### 起点往往比想象中简单

最开始我只是想「让它在群里回我一句话」。

结果一路挖下去：@提及判定、`bot_id` 缺失、发送谎报成功、附件丢弃、引用消息、鉴权换 URL……**每一个坑都牵出下一个坑。**

如果一开始就知道要改这么多，可能就不会动手了。但正因为是一步步走过来的，每一步都不算太难——难的是不知道还有多少步。

### 调试靠的是看数据，不是靠想

八轮修复里，真正靠「想」解决的有三轮，剩下五轮全靠**打日志看真实数据**：

- 第 3 轮：加了 `inbound body shape` 日志，才发现 `extracted=0`
- 第 5 轮：打印提取到的媒体详情，才确认伪元素匹配不上
- 第 7 轮：打印完整 HTTP 响应，才发现字段名是 `realUrl` 不是 `data.url`

**我一开始总是想「应该是什么样」，结果全是错的。** 只有看到真实数据，才知道问题在哪。

### 官方原版的问题，是投入问题不是能力问题

零测试、创建即停更、两个月无维护——这种状态下，任何边界情况都不会被发现。

我踩的每一个坑（@提及、`bot_id`、发送谎报成功、引用消息），上游全都有，只是没人测出来。

**所以有时候，fork 一份自己维护，比等官方修要快。**

---

## 参考

- [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
- [YuanbaoTeam/dsh-yuanbao](https://github.com/YuanbaoTeam/dsh-yuanbao)（上游）
- [anxi78/dsh-yuanbao](https://github.com/anxi78/dsh-yuanbao)（本分支）
- [@anxi78/dsh-yuanbao](https://www.npmjs.com/package/@anxi78/dsh-yuanbao)（npm）
- [YuanbaoTeam/yuanbao-openclaw-plugin](https://github.com/YuanbaoTeam/yuanbao-openclaw-plugin)（参考实现）
