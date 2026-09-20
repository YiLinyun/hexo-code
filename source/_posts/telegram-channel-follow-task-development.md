---
title: Telegram 频道关注任务开发记录
date: 2026-09-20 15:30:00
categories:
  - 后端开发
tags:
  - Telegram
  - Telegram Bot
  - Java
  - Webhook
  - Bot API
  - 任务系统
---

最近在项目里做了一个 Telegram 频道关注任务，整体需求其实比较简单：

> 用户从 App 跳转到 Telegram，完成账号绑定并关注指定频道，关注成功后完成任务并发放奖励。

一开始这个需求是按照“加入 Telegram 群组”来设计的，后面改成了“关注 Telegram 频道”。

实际开发下来发现，两者在 Bot API 上很多逻辑都可以复用，只需要针对成员状态和 Webhook 做一些处理。

## 整体流程

目前实现的流程大概如下：

```text
App 生成 Bot 链接
        ↓
打开 Telegram Bot
        ↓
自动发送 /start 参数
        ↓
绑定 Telegram ID 和 Pockly 用户 ID
        ↓
检查用户是否已经关注频道
        ↓
    ┌───────────┐
    │           │
   已关注       未关注
    │           │
    ↓           ↓
完成任务      Bot 返回频道链接
发放奖励          ↓
               用户关注频道
                    ↓
             chat_member 回调
                    ↓
             判断是否新加入
                    ↓
                 完成任务
                 发放奖励
```

除此之外，App 端还可以主动调用接口，由服务端通过 Telegram Bot API 查询用户当前是否已经关注频道。

所以目前实际上有两条判断路径：

```text
chat_member Webhook
```

以及：

```text
getChatMember 主动查询
```

两者结合使用。

------

## 使用 `/start` 完成账号绑定

App 首先生成一个带参数的 Bot 链接：

```text
https://t.me/<bot_username>?start=<bindToken>
```

例如：

```text
https://t.me/xxx_bot?start=xxxxxxxx
```

用户打开 Telegram 并点击 `Start` 后，Bot 会收到一条类似下面的消息：

```text
/start xxxxxxxx
```

Webhook 中可以通过：

```text
message.from.id
```

拿到用户的 Telegram ID。

然后再根据 `/start` 后面的参数找到对应的 App 用户。

最终建立：

```text
Pockly userId ↔ Telegram userId
```

这样的绑定关系。

这个绑定完成之后，后续无论是判断频道关注状态，还是增加其他 Telegram 相关任务，都可以继续复用。

------

## 判断用户是否已经关注频道

账号绑定完成之后，服务端调用 Telegram 的：

```text
getChatMember
```

接口查询用户在频道中的成员状态。

常见状态包括：

```text
creator
administrator
member
left
kicked
```

对于当前业务，只要用户状态是：

```text
creator
administrator
member
```

就可以认为用户当前已经关注频道。

Java 判断类似：

```java
String status = chatMember.getString("status");

return "creator".equals(status)
        || "administrator".equals(status)
        || "member".equals(status);
```

如果已经关注：

```text
直接完成任务
    ↓
发放奖励
```

如果没有关注：

```text
Bot 返回 Telegram 频道链接
    ↓
引导用户关注频道
```

------

## 使用 `chat_member` 监听关注状态

将机器人设置成频道管理员之后，可以通过：

```text
chat_member
```

Update 监听频道成员状态变化。

例如用户刚刚关注频道时，大概会收到这样的 Webhook：

```json
{
  "chat_member": {
    "chat": {
      "id": -1001234567890,
      "type": "channel"
    },
    "old_chat_member": {
      "status": "left"
    },
    "new_chat_member": {
      "user": {
        "id": 987654321
      },
      "status": "member"
    }
  }
}
```

这里最主要关注两个状态：

```text
old_chat_member.status = left
new_chat_member.status = member
```

也就是：

```text
left → member
```

可以认为用户刚刚关注了频道。

这时通过：

```text
new_chat_member.user.id
```

获取发生状态变化的 Telegram 用户 ID。

然后根据之前建立的：

```text
Telegram ID ↔ Pockly ID
```

绑定关系找到对应用户，完成任务并发放奖励。

------

## `from.id` 和 `new_chat_member.user.id` 的区别

这里有一个开发过程中比较容易理解错的地方。

在 `chat_member` Update 中：

```text
chat_member.from.id
```

并不一定代表成员本人。

它表示的是：

> 导致或者执行这次成员状态变化的人。

而真正发生状态变化的用户应该通过：

```text
new_chat_member.user.id
```

获取。

普通用户自己加入频道时，两者通常是相同的，所以很容易误以为可以直接使用 `from.id`。

但是如果以后出现管理员审批、Join Request 等场景，两者就可能不一致。

因此业务代码中统一使用：

```text
new_chat_member.user.id
```

会更加稳妥。

------

## 为什么 Webhook 和主动查询都要保留

如果只依赖：

```text
chat_member
```

会存在一个问题。

比如用户在做这个任务之前，就已经关注过 Telegram 频道。

那么之后打开任务时，他不会再次产生：

```text
left → member
```

事件。

如果系统只依赖 Webhook，这类用户就没有办法完成任务。

所以最终实现是：

### `chat_member`

负责实时监听：

```text
用户关注频道
用户取消关注频道
```

### `getChatMember`

负责主动确认：

```text
用户当前到底有没有关注频道
```

因此即使用户之前就已经关注频道，或者某次 Webhook 没有正常处理，也可以通过主动查询完成最终状态确认。

简单来说就是：

```text
chat_member
    ↓
监听“发生了什么”

getChatMember
    ↓
确认“现在是什么状态”
```

------

## 奖励幂等

Telegram 频道本身允许用户不断：

```text
关注
 ↓
取消关注
 ↓
重新关注
 ↓
再次取消
```

因此服务端可能不断收到：

```text
left → member
member → left
left → member
member → left
```

这样的 Webhook。

所以不能简单地把：

```text
收到 left → member
```

直接等同于：

```text
发放一次奖励
```

当前 Telegram 关注任务本身属于 `once` 类型任务。

真正发放奖励之前，还会检查该用户当前任务的完成和领奖状态。

同一个用户：

```text
同一个 once 任务
```

只允许完成并领取一次奖励。

因此即使 Telegram 重复产生回调，也不会重复发放奖励。

------

## Webhook 防刷

Telegram 的 `message` Update 基本可以理解为：

> 用户每向 Bot 发送一条消息，就可能产生一次新的 Webhook。

例如用户不断给 Bot 发送：

```text
你好
123
aaaa
test
```

都会产生新的消息事件。

而当前业务实际上只需要处理：

```text
/start xxx
```

所以普通消息没有必要进入后续业务逻辑。

可以直接过滤：

```java
String text = message.getText();

if (text == null || !text.startsWith("/start ")) {
    return;
}
```

这样即使有人不断给 Bot 发送普通消息，也不会触发数据库查询、任务处理、奖励发放等业务。

除此之外，Webhook 还可以校验 Telegram 请求携带的：

```text
X-Telegram-Bot-Api-Secret-Token
```

避免外部直接伪造 Telegram Webhook 请求。

如果需要进一步加强，还可以增加：

```text
update_id 幂等
Telegram ID 限流
bindToken 有效期
bindToken 一次性使用
```

这些限制。

------

## 三种 Telegram Update 的职责

这次开发里主要接触到了三个 Update 类型：

### `message`

主要处理用户给 Bot 发送的消息。

当前业务主要用于：

```text
/start bindToken
```

完成：

```text
Pockly 用户 ↔ Telegram 用户
```

身份绑定。

------

### `chat_member`

用于监听其他成员的状态变化。

例如：

```text
left → member
```

代表用户加入群组或者关注频道。

```text
member → left
```

代表用户退出群组或者取消关注频道。

------

### `my_chat_member`

这个和 `chat_member` 很容易混淆。

`my_chat_member` 指的是：

> Bot 自己在某个群或者频道里的成员状态变化。

例如：

```text
Bot 被加入频道
Bot 被设置为管理员
Bot 被移除
Bot 的管理员权限发生变化
```

所以当前“用户关注频道任务”的核心业务主要使用：

```text
message
+
chat_member
```

------

## 最终结构

整个 Telegram 频道关注任务最后可以拆成三个核心能力。

### `/start`

负责：

```text
App 用户 ↔ Telegram 用户
```

账号绑定。

### `getChatMember`

负责：

```text
主动确认用户当前是否关注频道
```

### `chat_member`

负责：

```text
实时监听用户关注 / 取消关注行为
```

最后再通过任务系统自身的：

```text
once
```

任务机制保证奖励不会重复发放。

整体实现本身并不复杂。

真正比较容易踩坑的地方，主要还是 Telegram Bot API 里面：

```text
message
chat_member
my_chat_member
```

三种 Update 的区别，以及：

```text
from.id
new_chat_member.user.id
```

这些字段实际代表的含义。

把这些关系理清之后，整体业务逻辑就比较清晰了。

这套账号绑定和成员状态校验逻辑后面也可以继续复用，如果以后增加 Telegram 群组任务、多频道关注任务或者其他 Telegram 活动，基本不需要重新设计整个流程。
