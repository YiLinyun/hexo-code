---
title: Firebase FCM 推送触达率只有 1%？一次被 Received 指标误导的踩坑记录
date: 2026-08-10 16:55:00
categories:
  - Android
  - Firebase
tags:
  - FCM
  - Firebase
  - Android
  - BigQuery
  - Push Notification
  - 消息推送
description: 记录一次 FCM 推送触达率异常的排查过程：每天发送 5 万多条消息，Firebase Console 的 Received 却只有 300 多。排查 Token、Priority、通知权限、客户端埋点均无果，最终通过 BigQuery 的 MESSAGE_DELIVERED 发现实际送达量达到 3.65 万，真正的问题不是 FCM，而是我们误解了控制台指标。
---

最近终于把一个困扰我们很久的 FCM 推送问题搞清楚了。

整个问题最开始看起来非常严重：

```text
每天 FCM 推送：50000+
Firebase Console Received：300+
```

如果直接按照 Firebase Console 中的 `Received` 来理解：

```text
300 / 50000 ≈ 0.6%
```

也就是说，我们的 FCM 推送触达率甚至不到 **1%**。

这已经不是“触达率有点低”的问题了，而是接近：

> 推送系统基本不可用。

于是我们围绕 FCM、Android、Token、通知权限、后台限制等方向排查了很长时间。

期间甚至已经出现了非常明显的数据矛盾：

```text
Firebase Received：300+
我们自己的通知点击埋点：500+
```

但即便这样，我们依然首先怀疑的是：

> 是不是自己的埋点错了？

而不是：

> Firebase Console 这个 Received 到底是不是我们理解的“设备触达”？

直到最后开启了 Firebase Cloud Messaging 的 BigQuery 数据导出。

查询之后发现：

```text
MESSAGE_DELIVERED：36500+
```

到这里才终于确定：

**我们的 FCM 推送本身根本没有之前想象中那么严重的问题。**

真正把整个排查方向带偏的，是 Firebase Console 中的 `Received` 指标。

这篇文章记录一下整个踩坑过程。

<!-- more -->

# 一、问题现象

我们的 App 使用 Firebase Cloud Messaging，也就是 FCM 进行消息推送。

服务端每天发送量大约：

```text
50000+
```

但 Firebase Console 中看到的 `Received` 非常低：

```text
300+
```

从数据上看：

```text
发送：50000+
Received：300+
```

粗略计算：

```text
300 / 50000 ≈ 0.6%
```

触达率不到 1%。

当时我们对 `Received` 的理解非常直接：

```text
Sent
    ↓
FCM
    ↓
Received
    ↓
用户设备
```

也就是说，我们默认：

> `Received` 基本就代表消息已经触达到设备。

既然 5 万多条消息只有 300 多条 Received，那么问题自然就变成：

> 为什么 FCM 有 99% 左右的消息没有成功送到用户设备？

然后整个排查就从这个前提出发了。

---

# 二、第一轮排查：是不是 Token 有问题？

首先怀疑的是 FCM Token。

这是推送问题里比较常见的一个方向。

比如：

- 用户卸载 App
- Token 已经失效
- Token 刷新后服务端没有及时更新
- 数据库中存在大量历史 Token
- Token 与用户绑定关系异常

服务端也重点关注了类似：

```text
UNREGISTERED
INVALID_ARGUMENT
QUOTA_EXCEEDED
UNAVAILABLE
```

等错误。

但实际排查结果并没有发现足以解释这个数据的问题。

如果每天发送 5 万条消息，真正只有 300 多条能到设备，那么理论上服务端应该能观察到非常明显的大规模异常。

但实际上并没有。

FCM API 的发送结果整体看起来是正常的。

于是继续往 Android 客户端方向排查。

---

# 三、怀疑 Android 后台限制

FCM 在 Android 上会受到不少系统因素影响，例如：

- Doze
- 后台运行限制
- 省电策略
- App 进程被杀
- 网络状态
- 厂商 ROM 后台限制

于是我们开始怀疑：

> 会不会消息虽然成功提交给 FCM，但是因为 Android 系统的后台限制，大量消息没有及时送达到设备？

尤其是 Android 各种后台策略一直都是消息推送里比较让人头疼的问题。

所以这一阶段重点检查了：

```text
FCM
 ↓
Android 系统
 ↓
App
```

这一段链路。

---

# 四、将消息 Priority 调整为 HIGH

为了尽可能减少后台限制导致的消息延迟，我们把 Android 消息优先级调整成了：

```json
{
  "android": {
    "priority": "high"
  }
}
```

也就是：

```text
HIGH
```

当时的预期是：

如果大量消息真的是因为 Doze 或后台调度导致没有及时下发，那么调整 Priority 之后，Firebase Console 中的 `Received` 应该至少会有一定程度的改善。

结果：

**几乎没有变化。**

还是：

```text
发送：50000+
Received：300+
```

到这里问题已经开始变得奇怪了。

但我们当时的第一反应依然是：

> 肯定还有某个地方没有排查到。

而不是 Firebase 的统计有问题。

---

# 五、开始自己做埋点验证

因为 Firebase Console 显示出来的触达率实在太离谱，我们决定不再只看 Firebase，而是自己增加客户端埋点。

主要增加了两组数据。

---

## 1. 统计目标用户的通知权限

第一组统计：

> 接收到推送的目标用户中，到底有多少人开启了 App 通知权限？

因为如果绝大部分用户都关闭了通知权限，那么即使消息能够到达设备，用户最终也可能看不到通知。

每天实际涉及的推送用户大约有：

```text
20000+
```

其中通知权限处于开启状态的用户有：

```text
10000+
```

也就是说：

每天至少有一万多目标用户，本身是允许 App 展示通知的。

这个数据虽然不能直接证明：

```text
FCM 消息一定成功到达设备
```

但至少说明：

> 问题不可能简单解释成“绝大多数用户关闭了通知权限”。

否则没办法解释为什么有一万多用户本身是允许展示通知的，而 Firebase Console 却显示一天只有：

```text
Received：300+
```

这时候数据已经有点对不上了。

但还不能证明 Firebase 有问题。

于是我们继续增加第二个埋点。

---

# 六、第二组埋点：统计用户点击通知进入 App

第二组埋点更加直接。

我们统计：

> 用户通过点击通知栏消息进入 App 的次数。

最后统计出来的点击率大约是：

```text
1%
```

按照每天：

```text
50000+
```

条推送来计算，一天大概能够观察到：

```text
500+
```

次通知点击。

然后一个非常明显的矛盾出现了。

Firebase Console 告诉我们：

```text
Received：300+
```

而我们的客户端埋点告诉我们：

```text
通知点击：500+
```

也就是：

```text
Firebase 认为收到消息的数量：300+

实际点击通知进入 App 的数量：500+
```

这在我们当时对 `Received` 的理解下，逻辑上其实是不成立的。

正常链路应该是：

```text
FCM 消息到达设备
        ↓
系统展示通知
        ↓
用户看到通知
        ↓
用户点击通知
        ↓
进入 App
```

所以理论上一定应该满足：

```text
点击数量 <= 收到数量
```

但我们的实际数据却是：

```text
点击数量 > Firebase Received
```

如果 `Received` 真的是“设备实际收到消息数”，那用户怎么可能点击一个 Firebase 认为根本没有收到的通知？

这是整个排查过程中非常关键的一个矛盾。

---

# 七、但我们依然第一时间怀疑自己

现在回头来看，这其实是整个问题最有意思，也最值得记录的地方。

当：

```text
Firebase Received：300+
```

而：

```text
我们自己的点击埋点：500+
```

的时候，其实已经非常应该怀疑：

> Firebase Console 这个 Received 到底统计的是什么？

但是我们当时完全没有这么想。

我们的第一反应是：

> 是不是自己的埋点有问题？

于是又开始排查自己的统计逻辑：

- 点击事件是不是重复触发了
- 埋点是不是重复上报
- 用户是不是重复点击
- SQL 聚合方式是不是有问题
- 数据有没有跨天
- Push 是否混入其他消息来源
- 客户端统计逻辑是不是写错了
- 用户和设备的统计维度是不是没对齐
- 通知权限统计是不是存在误差

因为在我们潜意识里：

```text
Firebase 官方数据
```

和：

```text
我们自己写的埋点
```

如果发生冲突，我们会自然地认为：

```text
Firebase 是对的
我们是错的
```

这也是整个问题真正把我们困住的地方。

---

# 八、为什么我们一直没有怀疑 Firebase Received？

还有一个重要原因。

`Received` 这个名字本身太有迷惑性了。

你看到：

```text
Sent
Received
```

第一反应非常容易理解成：

```text
Sent = 发送
Received = 收到
```

也就是：

```text
发送数量
    ↓
设备实际收到数量
```

而且我们当时通过各种渠道搜索相关资料时，得到的信息也很容易进一步强化这种理解。

所以整个排查过程中，我们几乎一直默认：

```text
Received = FCM 实际触达到设备
```

这个前提是没有问题的。

于是排查路线就变成了：

```text
Received 为什么这么低？
        ↓
是不是 Token 有问题？
        ↓
是不是 FCM 配置有问题？
        ↓
是不是 Priority 有问题？
        ↓
是不是 Doze？
        ↓
是不是 Android 后台限制？
        ↓
是不是通知权限？
        ↓
为什么自己的埋点和 Firebase 对不上？
        ↓
是不是自己的埋点又有问题？
```

我们一直在努力解释：

> 为什么自己的所有数据都无法佐证 Firebase 的 300+ Received？

却从来没有真正回头检查最开始的那个前提：

> `Received` 到底是不是我们理解的那个“真实触达”？

这是整个排查链路中最大的坑。

---

# 九、真正的转折点：开启 BigQuery

最后，我们决定开启 Firebase Cloud Messaging 的 BigQuery 数据导出。

Firebase 开启 BigQuery Export 之后，可以查看更加详细的消息事件。

其中我们重点关注一个事件：

```text
MESSAGE_DELIVERED
```

等 BigQuery 开始产生数据之后，我们查询了一天的数据。

结果看到：

```text
MESSAGE_DELIVERED ≈ 36500
```

这个数字一出来，之前所有奇怪的现象一下子都解释通了。

实际情况并不是：

```text
发送：50000+
        ↓
设备收到：300+
```

而更接近：

```text
发送：50000+
        ↓
FCM
        ↓
MESSAGE_DELIVERED：36500+
        ↓
设备
```

粗略计算：

```text
36500 / 50000 ≈ 73%
```

也就是说，从 BigQuery 的消息投递数据来看：

**我们的实际设备级送达情况是在 70%+ 这个量级。**

而不是 Firebase Console 给我们造成的：

```text
不到 1%
```

的印象。

---

# 十、这时候之前的所有矛盾都解释通了

回过头看之前的数据：

每天推送：

```text
50000+
```

涉及用户：

```text
20000+
```

其中开启通知权限：

```text
10000+
```

通知点击：

```text
500+
```

BigQuery：

```text
MESSAGE_DELIVERED：36500+
```

这些数据实际上是在同一个合理范围里的。

为什么有大量用户能够点击通知？

因为确实有大量消息已经送到了设备。

为什么业务上没有明显感觉 99% 用户收不到消息？

因为实际也根本没有 99% 的消息丢失。

为什么我们自己测试的时候 FCM 基本是正常的？

因为 FCM 本身没有出现我们想象中那么严重的问题。

真正和其他所有数据完全对不上的，只有：

```text
Firebase Console Received：300+
```

---

# 十一、MESSAGE_DELIVERED 可以帮助确认什么？

整个 FCM 链路可以简单理解成：

```text
业务服务器
    ↓
FCM Backend
    ↓
Google / Android 消息传输
    ↓
设备上的 FCM SDK
    ↓
App
    ↓
系统通知栏
    ↓
用户看到通知
```

BigQuery 中的：

```text
MESSAGE_DELIVERED
```

可以用来判断消息已经进入设备侧的 FCM 投递链路。

所以如果我们的目标是排查：

> FCM 到底有没有把消息送到设备？

那么：

```text
MESSAGE_DELIVERED
```

显然比单纯盯着 Firebase Console 的 `Received` 更有排查价值。

我们最终观察到：

```text
发送：50000+
MESSAGE_DELIVERED：36500+
```

已经足以证明：

> FCM 推送链路并不存在“5 万条只触达 300 条”这种级别的问题。

这对于我们来说其实已经完成了最重要的一步。

因为我们最开始排查的目标就是确认：

> 到底是不是我们的推送系统本身出了严重问题？

BigQuery 的数据最终证明：

**不是。**

---

# 十二、为什么 Firebase Console Received 会和 BigQuery 差这么多？

这也是这次最容易误解的地方。

我们之前一直默认：

```text
Firebase Console Received
```

就是：

```text
FCM 真实设备送达数量
```

但实际上 Firebase Console 中展示的 Messaging 数据存在自己的统计口径。

尤其如果使用的是：

```text
data message
```

还需要特别注意 Firebase Analytics、Analytics Label 等相关统计条件。

比如服务端发送的数据可能类似：

```json
{
  "message": {
    "token": "xxxxx",
    "data": {
      "title": "title",
      "body": "body"
    },
    "android": {
      "priority": "high"
    }
  }
}
```

这种情况下，Firebase Console 中看到的统计数据并不能简单等价成：

> FCM 实际送达设备的完整数量。

这也是我们之前一直误判问题严重程度的核心原因。

---

# 十三、BigQuery 查询示例

开启 Firebase Cloud Messaging BigQuery Export 之后，可以按照事件进行统计。

最简单的查询：

```sql
SELECT
    event,
    COUNT(*) AS count
FROM
    `your_project.firebase_messaging.data`
WHERE
    DATE(event_timestamp) = '2026-08-10'
GROUP BY
    event
ORDER BY
    count DESC;
```

可以查看不同 Event 的数量。

如果需要按照消息和设备进行去重，可以继续使用：

```sql
SELECT
    event,
    COUNT(
        DISTINCT CONCAT(message_id, instance_id)
    ) AS count
FROM
    `your_project.firebase_messaging.data`
WHERE
    DATE(event_timestamp) = '2026-08-10'
GROUP BY
    event
ORDER BY
    count DESC;
```

这样可以重点关注：

```text
MESSAGE_ACCEPTED
MESSAGE_DELIVERED
```

等事件。

---

# 十四、还可以按照 message_type 排查

如果 Firebase Console 和 BigQuery 的差异非常大，可以继续按照消息类型统计：

```sql
SELECT
    message_type,
    event,
    COUNT(
        DISTINCT CONCAT(message_id, instance_id)
    ) AS count
FROM
    `your_project.firebase_messaging.data`
WHERE
    DATE(event_timestamp) = '2026-08-10'
GROUP BY
    message_type,
    event
ORDER BY
    message_type,
    event;
```

如果自己的业务主要发送：

```text
DATA_MESSAGE
```

同时 BigQuery 中：

```text
MESSAGE_DELIVERED
```

数量非常正常，但 Firebase Console 的 `Received` 非常低，那么这时候就应该开始重点检查：

> 两边的统计口径是不是根本不同？

而不是第一时间继续怀疑 FCM 的实际投递链路。

---

# 十五、MESSAGE_DELIVERED 也不代表用户一定看到了通知

这里还需要额外说明一点。

虽然：

```text
MESSAGE_DELIVERED
```

可以帮助我们确认 FCM 到设备这一段没有之前想象中那么大的问题。

但：

```text
MESSAGE_DELIVERED
```

也不能简单理解成：

```text
用户一定看到了通知
```

因为后面还有一段客户端链路：

```text
FCM
    ↓
MESSAGE_DELIVERED
    ↓
设备 FCM SDK
    ↓
App 消息处理
    ↓
NotificationManager
    ↓
Android 通知栏
    ↓
用户看到
```

如果：

```text
MESSAGE_DELIVERED 很高
```

但是用户实际看到通知的比例依然很低，那么接下来应该排查的方向就完全不同了，例如：

- Android 13+ 通知权限
- 用户是否关闭 App 通知
- Notification Channel 是否关闭
- App 是否自己过滤消息
- `FirebaseMessagingService` 消息处理
- Notification 创建逻辑
- 前后台状态差异
- 厂商 ROM 的通知展示限制
- 业务代码是否主动丢弃某些消息

这时候问题已经不是：

```text
FCM 有没有把消息送下来？
```

而是：

```text
消息到设备以后，有没有真正展示给用户？
```

一定要把这两个阶段拆开分析。

---

# 十六、整个问题最大的坑：我们过于相信 Firebase Console

现在重新回看整个排查过程，真正浪费我们最多时间的，其实不是 FCM 有多复杂。

而是：

> **我们过于相信 Firebase Console 展示出来的数据。**

当 Firebase 告诉我们：

```text
Sent：50000+
Received：300+
```

时，我们几乎没有怀疑地接受了这个前提：

```text
真实触达只有 300+
```

然后开始围绕这个结论排查所有东西。

但是随着排查继续，自己的数据其实一直在告诉我们：

> 这个结论可能不对。

第一组数据：

```text
每天推送用户：20000+
通知权限开启：10000+
```

无法解释为什么实际触达只有 300。

第二组数据更加夸张：

```text
Firebase Received：300+
自己统计通知点击：500+
```

按照我们当时对 `Received` 的理解：

这件事情在逻辑上根本不应该发生。

一个用户需要经历：

```text
收到消息
    ↓
看到通知
    ↓
点击通知
```

才能产生点击事件。

所以：

```text
点击数量 > 收到数量
```

本身就是一个非常强烈的异常信号。

但我们当时的第一反应仍然是：

```text
自己的埋点有问题
```

而不是：

```text
Firebase 的统计口径可能和我们想的不一样
```

因为 Firebase 是官方平台。

因为控制台的数据看起来更加权威。

也因为 `Received` 这个名字非常容易让人理解成：

```text
设备实际收到消息
```

甚至我们通过其他渠道查询到的信息，也一直在强化这种理解。

所以整个排查过程实际上变成了：

```text
先相信 Firebase 的 300+
        ↓
发现自己的数据不一致
        ↓
怀疑自己的代码
        ↓
修改
        ↓
继续不一致
        ↓
继续怀疑自己
```

我们一直在尝试：

> 用自己的数据解释 Firebase 为什么是对的。

却没有反过来问：

> Firebase Console 这个数据本身，到底能不能用来回答我们现在的问题？

直到 BigQuery 中出现：

```text
MESSAGE_DELIVERED：36500+
```

才终于打破了这个前提。

---

# 十七、如果重新排查一次，我会怎么做？

以后再遇到类似：

```text
FCM 触达率异常
```

的问题，我不会再只围绕 Firebase Console 的 `Received` 排查。

我会按照下面这个顺序。

## 第一步：确认服务端发送是否正常

先检查 FCM API 的发送结果。

重点关注：

```text
UNREGISTERED
INVALID_ARGUMENT
QUOTA_EXCEEDED
UNAVAILABLE
```

等异常。

如果存在大量 Token 失效，先解决 Token 生命周期问题。

---

## 第二步：确认自己的业务指标

客户端最好有自己的埋点。

至少可以统计：

```text
目标用户数量
通知权限开启比例
通知展示
通知点击
App 打开
```

这些指标可以形成自己的业务观察链路。

不要完全依赖第三方平台上的一个数字。

---

## 第三步：如果数据出现逻辑矛盾，先检查指标定义

比如：

```text
Received：300
通知点击：500
```

这种情况下不要只想着：

```text
是不是自己的埋点错了？
```

还应该同时问：

```text
Received 的技术定义到底是什么？
统计条件是什么？
统计范围是什么？
哪些消息会进入这个指标？
```

指标名字只能帮助理解。

不能代替指标定义。

---

## 第四步：尽早开启 BigQuery Export

如果需要认真分析 FCM 的设备投递情况，建议尽早开启：

```text
Firebase Cloud Messaging BigQuery Export
```

重点观察：

```text
MESSAGE_ACCEPTED
MESSAGE_DELIVERED
```

而不是等问题出现很久之后再开启。

这次如果我们一开始就有 BigQuery 数据，整个问题可能很快就结束了。

---

## 第五步：把“送达到设备”和“用户看到通知”拆开

最终整个链路最好分成两部分：

```text
第一段：

服务端
  ↓
FCM
  ↓
设备

第二段：

设备
  ↓
App
  ↓
通知栏
  ↓
用户点击
```

第一段出现问题，就排查：

```text
Token
FCM
网络
消息 Priority
投递事件
```

第二段出现问题，则排查：

```text
通知权限
Notification Channel
客户端处理
Android 系统限制
厂商 ROM
业务逻辑
```

不要把两类问题混在一起。

---

# 十八、最终结论

这次问题最后几个核心数据是：

```text
每天 FCM 发送：

50000+

Firebase Console Received：

300+

自己的通知点击埋点：

500+

BigQuery MESSAGE_DELIVERED：

36500+
```

如果只相信 Firebase Console：

```text
FCM 触达率 < 1%
```

整个推送系统看起来几乎已经不可用。

但 BigQuery 最终告诉我们：

```text
36500 / 50000 ≈ 73%
```

实际情况根本不是一个数量级。

因此这次最终能够确认：

> **我们的 FCM 推送链路本身并不存在最开始认为的严重触达问题。**

真正的问题是：

> **我们误把 Firebase Console 的 Received 当成了完整、准确的实际设备触达数量。**

这是这次踩坑最大的教训。

---

# 写在最后

作为开发人员，我们通常很容易相信平台提供的数据。

尤其是：

```text
Firebase
AWS
Google Cloud
第三方监控平台
各种 Dashboard
```

这些平台的数据往往天然带着一种“官方就是正确答案”的感觉。

但其实：

**数据可以是准确的，理解仍然可以是错误的。**

一个指标叫：

```text
Received
```

不代表它就一定等于：

```text
你脑海里理解的那个 Received
```

当不同数据之间发生明显冲突时：

不要只是怀疑：

```text
我的代码是不是错了？
我的埋点是不是错了？
我的 SQL 是不是错了？
```

还应该反过来检查：

```text
这个第三方指标到底是怎么定义的？
```

这次我们最大的教训不是某个 FCM 参数怎么配置。

而是：

> **先理解指标，再相信指标。**

否则系统可能根本没有坏。

真正把排查方向带偏的，只是一个看起来非常合理的数据。