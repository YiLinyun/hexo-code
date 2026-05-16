# 推荐视频推送链路：从 Quartz 扫描到 RocketMQ 延迟投递，再到 FCM 触达

这次做的不是“发一条推送”这么简单，而是把推荐视频推送拆成了一条稳定链路：

`recommendedVideoPushJob -> RecommendVideoPushProducer -> RecommendVideoPushConsumer -> RecommendVideoPushService -> FCM`

核心目标就一句话：

> 把“合适的视频”在“合适的本地时间”推给“合适的用户”，同时尽量别重复、别打扰、别把系统自己搞炸。

---

## 一、为什么这事不能直接定时查库然后发

一开始最容易想到的写法是：

1. Quartz 定时跑一遍
2. 扫用户
3. 满足条件就直接调 FCM

这个思路看起来直给，实际上问题一堆：

- 用户分布在不同时区，服务器时间不等于用户时间
- 推送窗口有午间、晚间，不是“查到就发”
- 同一个用户不能被重复塞进队列，更不能被重复推送
- 推荐视频是动态列表，还得跳过已经播放过的
- 图片要先转存 CDN，FCM 发的不是一串字，是一条能点、能看、能落地的消息

所以最后我把它拆成了“任务只负责排期，消费才负责执行”的两段式架构。这样系统不需要每分钟硬发消息，只需要提前把消息安排好，到点再消费。

---

## 二、整体架构

```mermaid
flowchart LR
    A["Quartz Job<br/>recommendedVideoPushJob"] --> B["预过滤<br/>Redis MQ锁"]
    B --> C["查用户资料<br/>FCM Token / IANA 时区 / 新用户过滤"]
    C --> D["计算下次投递时间<br/>国家窗口 + 用户打散"]
    D --> E["RocketMQ 延迟消息<br/>RecommendVideoPushProducer"]
    E --> F["RocketMQ Consumer<br/>RecommendVideoPushConsumer"]
    F --> G["再次校验时间窗口"]
    G --> H["取未播放推荐视频<br/>Redis 推荐池 + 播放记录"]
    H --> I["图片转 CDN + vid 加密"]
    I --> J["FCM 推送"]
    J --> K["释放锁并安排下一次投递"]
```

这条链路里我刻意做了三层分工：

- `Job` 负责“找人”和“排期”
- `Producer / Consumer` 负责“解耦执行时机”
- `Service` 负责“所有业务判断”

这么拆的好处很实在：定时任务不碰发送细节，MQ 不碰业务规则，业务逻辑不散落在各处。

---

## 三、主链路是怎么跑起来的

### 1. Job 先扫推荐池，但不会一上来就查全量用户

`RecommendedVideoPushJob` 的入口很克制，先从 Redis 推荐池拿用户 ID，而不是上来全表扫用户。[RecommendedVideoPushJob.java](/D:/code/video_tuber_admin/videoTuber-admin/src/main/java/com/muni/web/job/RecommendedVideoPushJob.java:44)

推荐池 key 是：

- `recommend:video`

也就是说，只有“已经有推荐视频可发”的用户才会进入这条链路。这个切法很值，至少避免了把推送系统做成数据库压力制造机。

### 2. 先抢一把短锁，做预过滤

真正让我觉得这版实现开始像“工程方案”而不是“业务脚本”的地方，是这把预过滤锁。

Job 会先给每个用户抢一个短期 MQ 锁，默认 30 分钟。抢不到就说明这个用户大概率已经在队列里了，直接跳过。[RecommendedVideoPushJob.java](/D:/code/video_tuber_admin/videoTuber-admin/src/main/java/com/muni/web/job/RecommendedVideoPushJob.java:24)

这一步的意义不是防并发炫技，而是防止：

- 同一个用户反复查库
- 同一个用户反复塞延迟消息
- Quartz 每轮都对老用户重复做无意义计算

说白了，先用 Redis 把“明显没必要处理的人”挡在门外。

### 3. 过筛用户：新用户、FCM Token、IANA 时区，一个都不能少

预过滤后，Job 才会查用户详情，然后走 `shouldPush`：

- 新用户首日不推
- 没有 FCM Token 不推
- 没有 IANA 时区不推

这部分逻辑在 `RecommendVideoPushService` 里统一收口，而不是散在 Job/Consumer 里，这点非常重要。[RecommendVideoPushService.java](/D:/code/video_tuber_admin/videoTuber-system/src/main/java/com/muni/system/service/impl/RecommendVideoPushService.java:206)

这里最容易踩坑的是“新用户首日”的定义。代码里是按服务器系统时区比较创建日期和今天日期，不是按用户本地时区算。[RecommendVideoPushService.java](/D:/code/video_tuber_admin/videoTuber-system/src/main/java/com/muni/system/service/impl/RecommendVideoPushService.java:81)

这能跑，但要说绝对优雅，也没有。跨时区业务最后总会提醒你一句：

> 服务器今天，不一定是用户的今天。

### 4. 不直接发，而是算“下次应该发的时间”

通过校验后，系统不会立刻推，而是调用 `calculateNextDeliverTimestamp` 算出用户下一次应该被消费的时间点。[RecommendVideoPushService.java](/D:/code/video_tuber_admin/videoTuber-system/src/main/java/com/muni/system/service/impl/RecommendVideoPushService.java:114)

这个方法做了三件事：

- 根据用户 `IANA` 时区换算本地时间
- 根据国家决定推送窗口
- 根据用户 ID 做 10 分钟内打散，避免整点洪峰

国家窗口目前是：

- 印度：13:00-14:00、20:00-21:00
- 巴西：12:30-13:30、20:30-22:30
- 其他地区：12:00-13:00、19:00-20:00

配置放在 `RecommendVideoPushWindowConstants`，扩展新国家时基本不用碰主流程代码。[RecommendVideoPushWindowConstants.java](/D:/code/video_tuber_admin/videoTuber-common/src/main/java/com/muni/common/constant/RecommendVideoPushWindowConstants.java:12)

这里我觉得最关键的，不是“算时间”，而是“别把所有人都算到同一分钟”。  
所以加了：

- `WINDOW_SPREAD_MILLIS = 10 分钟`
- `userId hash` 打散投递时刻

这个改动很小，但对系统波峰波谷非常友好。否则整点一到，MQ、图片下载、CDN、FCM 一起抽风，锅还不一定知道该甩给谁。

### 5. Producer 只干一件事：发延迟消息

`RecommendVideoPushProducer` 很薄，核心就两件事：

- 校验投递时间必须大于当前时间
- 校验 RocketMQ 5.x 延迟消息最大只支持 24 小时

然后把 `userId` 作为消息体，把目标时间戳塞进 `deliveryTimestamp`。[RecommendVideoPushProducer.java](/D:/code/video_tuber_admin/videoTuber-admin/src/main/java/com/muni/web/mq/recommendVideo/RecommendVideoPushProducer.java:26)

我很喜欢这种“薄 Producer”设计。Producer 不负责聪明，只负责可靠。聪明都放在 Service，后面维护时脑子会轻松很多。

### 6. Consumer 到点消费，但还要再验一次窗口

消息到了消费者，不是立刻发 FCM，而是先再次判断用户当前是否处于推送窗口。[RecommendVideoPushConsumer.java](/D:/code/video_tuber_admin/videoTuber-admin/src/main/java/com/muni/web/mq/recommendVideo/RecommendVideoPushConsumer.java:70)

这一步看起来像重复判断，实际上非常必要：

- 延迟消息不代表绝对准点
- 用户时区、配置、窗口规则可能变化
- 临界时间点永远是生产事故的老朋友

如果当前不在窗口内，就不发，直接安排下一次投递。这个补偿设计让整条链路更像“可持续调度”，而不是“一锤子买卖”。

### 7. 真正发之前，再从推荐池里挑一条“还没播过”的视频

`findUnplayedVideo` 会从 Redis 推荐列表中，挑出 index 最小、且没有播放记录的视频。[RecommendVideoPushService.java](/D:/code/video_tuber_admin/videoTuber-system/src/main/java/com/muni/system/service/impl/RecommendVideoPushService.java:281)

这里有两个点挺实用：

- 播放过滤靠 `playback:{userId}:{vid}`
- 选中视频后，会从推荐列表里移除，避免下次还拿到同一条

这就把“推荐系统给了一堆候选”和“推送系统每次只发一条”衔接起来了。

---

## 四、两个锁，解决两个层面的重复问题

这套实现里最有意思的地方，不是 MQ 本身，而是锁的分层。

### 1. MQ 锁：防止重复入队

key:

- `recommend:lock:mq:{userId}`

用途：

- Job 预过滤时先占位
- 算出真实延迟后刷新过期时间
- Consumer 消费完成或异常时释放

它解决的是“同一个用户别在队列里排出一串分身”。

### 2. FCM 锁：防止短时间重复触达

key:

- `recommend:lock:fcm:{userId}`

用途：

- 真正调 FCM 之前加 2 小时锁

它解决的是“就算队列层面没重复，也别让用户在短时间内被连续骚扰”。

这两个锁分开非常合理。  
一个防系统重复工作，一个防用户重复挨打。职责不一样，千万别混。

---

## 五、推送真正落地时，麻烦其实不在 MQ，在素材

`sendFcmPush` 并不是拿着标题和 token 就发了，中间还做了两件很现实的脏活：

1. 把推荐图下载下来，再转存到 CDN
2. 把 `vid` 做加密后放进推送 data

图片处理在 `RecommendImageService`，它会：

- 先下载原图
- 算图片 MD5
- OSS 已存在就复用
- 不存在再上传

这个实现挺接地气，因为外链图最大的问题从来不是“能不能显示”，而是“不稳定、失效、慢、不可控”。  
推送素材如果直接吃源站图，线上迟早会教育你。

---

## 六、这套方案里我觉得最值钱的几个设计点

### 1. 任务负责排期，消费负责执行

把“谁该发”与“什么时候真发”拆开后，系统弹性一下就出来了。

### 2. 所有时间判断都围绕用户本地时区

不是按服务器时间粗暴统一发，而是按用户所在地区的午间/晚间窗口发。对触达体验来说，这是本质差异。

### 3. 通过 hash 打散，主动削峰

很多推送系统不是死在业务逻辑上，是死在“大家都在 20:00:00 一起发”。

### 4. 双锁设计，把“重复入队”和“重复触达”拆开治理

这比单锁硬扛靠谱得多，也更好排查问题。

### 5. Consumer 消费后继续安排下一次

这让链路从“定时批处理”变成了“持续调度循环”，系统能自己往后滚。

---

## 七、这次实现里最难受的几个点

### 1. 时区不是技术细节，是业务主逻辑

一旦涉及全球用户，“几点发”就不是一个 if/else，而是一整套本地时间模型。谁还把时区当边角料，谁就等着线上挨打。

### 2. 临界时间点非常烦

代码里专门对 `deliverTimestamp <= currentTime` 做了重算，还限制了重试次数。[RecommendVideoPushService.java](/D:/code/video_tuber_admin/videoTuber-system/src/main/java/com/muni/system/service/impl/RecommendVideoPushService.java:132)

原因很简单：你以为你算的是“下一分钟”，系统常常告诉你“已经过了”。

### 3. 延迟消息不是万能的

RocketMQ 5.x 延迟消息最大只支持 24 小时，所以 Producer 里必须显式校验。[RecommendVideoPushProducer.java](/D:/code/video_tuber_admin/videoTuber-admin/src/main/java/com/muni/web/mq/recommendVideo/RecommendVideoPushProducer.java:35)

很多时候不是业务想怎么排就怎么排，而是中间件先把天花板拍你脸上。

### 4. 图片外链真的很烦

推送图如果不转 CDN，后面不是超时就是 404，再不然是客户端展示异常。  
这部分技术含量不高，但不做就一定出事，典型的“脏但必须有人做”。

---

## 八、如果以后继续演进，我会优先做什么

### 1. 把国家窗口配置化

现在窗口常量写在代码里，够用，但还不够优雅。后面更适合抽到配置中心或数据库。

### 2. 新用户首日判断改成用户本地日期

当前按服务器日期判断，逻辑简单，但严格说不够“全球化正确”。

### 3. 给推送结果补可观测性

比如：

- 每日入队数
- 实际消费数
- FCM 成功率
- 图片转存失败率
- 每个国家/时区的触达表现

不然功能做完了，只能靠日志里刨真相，排障体验比较原始。

---

## 九、最后总结

这条推荐视频推送链路，本质上解决的是三个问题：

1. 谁值得推
2. 什么时候推
3. 怎么稳定地推到用户手上

从实现上看，它不是一个“发消息”的点功能，而是一条带时区、延迟调度、幂等控制、素材处理、推荐过滤的完整链路。

如果只看代码量，这功能不算夸张。  
但如果真把它放到线上语境里看，它的难点从来不是“调用 FCM”，而是：

> 怎么让推送既准时、又不重复、还别把自己系统推崩。

这部分做完之后，我对推送类需求最大的感受就是：

> 真正麻烦的从来不是“发出去”，而是“发得对”。
