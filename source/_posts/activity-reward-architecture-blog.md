---
title: 从 CRUD 到统一发奖与资产消耗：一次活动系统架构重构记录
date: 2026-06-16 10:30:00
categories:
  - 后端架构
tags:
  - Java
  - Spring Boot
  - MyBatis
  - 活动系统
  - 幂等
  - Redis
---

这次活动模块包含签到、任务、邀请奖励、抽奖、道具包、流水查询等一组接口。最开始设计时，我很自然地沿着传统 CRUD 的思路往下拆：签到接口自己判断今天是否签到、自己发积分；任务接口自己判断任务是否可领取、自己写任务记录、自己加金币；抽奖接口自己扣金币、自己随机奖品、自己发放奖励；道具接口自己扣库存、自己延长 VIP。

这种写法短期能跑，但很快会遇到一个问题：活动越来越多时，真正复杂的不是接口数量，而是“奖励怎么发”和“资产怎么扣”。如果每个接口都各写一套加积分、加 VIP、加道具、扣积分、扣道具、写流水、处理幂等的逻辑，代码会迅速变成一堆看起来相似、细节又不完全一致的业务分支。

所以后面我不断和 AI 沟通，把架构从“一个接口一套 CRUD”调整成了现在的模式：业务接口只负责编排，核心资产动作收敛到两个服务。

- `ActivityRewardGrantService`：统一处理发奖。
- `UserAssetConsumeService`：统一处理资产消耗。

抽奖、签到、任务、邀请奖励、道具使用都围绕这两个服务展开。接口层变薄了，复杂度被压到更稳定、更容易测试和复用的服务层里。

## 整体架构

新活动接口集中在 `ActivityController` 下，路径是 `/apiV2/activity`。控制器主要做四件事：

1. 通过 `RequestUtils.getPublicId()` 找到当前用户。
2. 做必要的参数校验，比如 `taskId`、`requestNo`、`changeType`。
3. 调用活动服务完成业务。
4. 记录请求和耗时日志，包装 `AjaxResult` 返回。

控制器没有直接操作积分、VIP、道具库存，也没有把发奖逻辑写在接口里。比如任务奖励接口最终调用的是：

```java
activityRewardGrantService.grantTaskReward(currentUser, taskId, taskDate)
```

抽奖接口调用的是：

```java
lotteryService.lotteryDraw(currentUser, requestNo)
```

而 `LotteryService` 内部也不直接写积分和道具，而是先调用消耗服务扣资产，再调用发奖服务发放奖励：

```java
userAssetConsumeService.consumeProp(...)
userAssetConsumeService.consumePoints(...)
activityRewardGrantService.grantGenericReward(...)
```

这样整个活动系统的分层大致是：

```text
ActivityController
  -> ActivityProfileService       查询个人中心、签到、道具包
  -> ActivityRewardGrantService   统一发奖
  -> UserAssetConsumeService      统一消耗
  -> LotteryService               抽奖编排
  -> PropUseService               道具使用编排
  -> InviteCodeService            邀请码绑定
```

这里最关键的变化是：签到、任务、抽奖、邀请不再各自实现一套“奖励落库逻辑”，而是把奖励抽象成统一的 `TaskRewardItem`。不管来源是任务奖励、签到奖励还是抽奖奖品，只要最终能表达成“奖励类型 + 数量 + 道具 ID + 单位”，就可以交给同一个发奖服务处理。

## 表结构设计

这套活动系统的表设计不是围绕接口设计的，而是围绕“配置、用户状态、业务记录、资产流水”来设计。

### 任务配置表：`vt_task`

`vt_task` 保存活动任务配置。任务不只是普通每日任务，也可以承载签到奖励、邀请奖励等活动配置。表里最重要的是任务类型、周期类型和奖励配置。

比如任务可以分成：

- 每日任务：每天可完成一次。
- 生命周期任务：整个用户生命周期只可完成一次。
- 活动任务：用于签到、邀请等特殊业务的奖励配置。

这样做的好处是，奖励规则可以配置化。接口不需要写死“第几天签到发多少积分”，而是可以通过任务配置找到对应奖励，再交给统一发奖服务。

### 用户任务记录表：`vt_user_task`

`vt_user_task` 是用户领取任务奖励后的业务记录。它记录用户、任务、奖励快照、任务日期等信息。

这里有一个非常重要的设计：用唯一索引保证任务领取幂等。

对于每日任务，唯一维度可以是：

```text
user_id + task_id + task_date
```

对于生命周期任务，`task_date` 可以为空或使用固定语义，由业务周期来约束。代码里在插入 `vt_user_task` 时捕获 `DuplicateKeyException`，如果并发请求同时进来，只有一个请求能插入成功，另一个请求会被唯一索引挡住，然后返回“奖励已领取”。

这个设计比“先查有没有，再决定要不要插入”更可靠。因为在并发场景下，两个请求可能同时查到没有记录，然后同时发奖。真正可靠的边界应该放在数据库唯一索引上。

### 签到记录表：`vt_user_checkin`

签到也不是只更新一个用户字段，而是单独保留签到记录。核心字段包括：

- `user_id`
- `checkin_date`
- `cycle_day`
- `total_checkin_days`
- `task_id`
- `reward_snapshot`
- `create_time`
- `update_time`

其中 `user_id + checkin_date` 应该有唯一索引，代码里也是先插入签到记录，再发奖励。这个顺序很关键：先让数据库确认“今天这次签到是唯一的”，再进入发奖链路。

签到成功后，再回写本次签到的周期天数、累计签到天数、关联任务 ID 和奖励快照。这样后续排查时，不需要重新推导当时发了什么，直接看签到记录即可。

### 道具配置表：`vt_prop`

`vt_prop` 是道具定义表。抽奖奖品里如果抽中道具，只记录 `propId`，真正的道具名称、类型、效果以 `vt_prop` 为准。

这个细节很有价值。早期很容易把奖品名、道具名散落在代码和配置里，最后出现“抽奖配置叫 vip_card_2h，但道具表叫 2小时VIP卡”的不一致。现在统一以道具表为准，发奖服务只关心 `propId`，展示层再按道具配置取名称。

### 用户道具表：`vt_user_prop`

`vt_user_prop` 保存用户拥有的道具数量。核心字段包括：

- `user_id`
- `prop_id`
- `quantity`
- `is_new`
- `expire_time`
- `create_time`

发放道具和消耗道具都会操作这张表。并发控制上，代码使用 `select ... for update` 锁定用户某个道具的行，再更新数量。

如果发奖时用户还没有这类道具，会先插入一条 `vt_user_prop`。这里同样需要依赖 `user_id + prop_id` 的唯一索引：并发发放同一个道具时，可能两个事务都发现没有记录，然后同时插入。代码捕获 `DuplicateKeyException` 后重新 `selectForUpdate`，再做数量累加。

### 统一流水表：`vt_ledger`

`vt_ledger` 是整个活动系统最核心的审计表。积分增加、积分消耗、VIP 增加、道具发放、道具消耗、勋章发放都会写流水。

关键字段包括：

- `user_id`：用户 ID。
- `prop_id`：如果是道具相关流水，记录道具 ID。
- `biz_type`：业务来源，比如 `task`、`checkin`、`lottery`、`invite`。
- `biz_id`：业务记录 ID，比如 `userTaskId` 或抽奖的 `requestNo`。
- `reward_type`：资产或奖励类型，比如 `point`、`prop`、`vip_time`、勋章类型。
- `change_type`：`add` 或 `sub`。
- `change_quantity`：变化数量。
- `balance_after`：变化后的余额。
- `vip_time_before` / `vip_time_after`：VIP 时间变动前后的快照。
- `create_time`：流水创建时间。

这张表解决了两个问题：

第一，用户资产变化可以追踪。出了问题可以查“这笔积分为什么加了”“这个道具什么时候扣了”。

第二，报表可以统一从流水聚合。活动报表不需要分别统计签到表、任务表、抽奖表、邀请表，而是可以按 `biz_type`、`reward_type`、`change_type` 聚合金币发放、金币消耗、道具发放等指标。

## 发奖服务：把奖励动作收敛成一个入口

`ActivityRewardGrantService` 是这套架构里最重要的服务之一。它提供两个核心入口：

```java
GrantRewardResult grantTaskReward(PocklyUser user, Long taskId, LocalDate taskDate)

GrantRewardResult grantGenericReward(PocklyUser user, String bizType, Long bizId, List<TaskRewardItem> rewardItems)
```

`grantTaskReward` 面向任务领取场景。它会读取任务配置，判断任务周期，插入 `vt_user_task`，然后调用内部的 `doGrantReward` 发放奖励。

`grantGenericReward` 面向通用发奖场景。抽奖、邀请、一些非任务来源的活动都可以直接构造 `TaskRewardItem`，然后调用这个方法。

内部真正的发奖逻辑会按奖励类型分发：

- `point`：更新用户积分，写 `vt_ledger`。
- `vip_time`：延长用户 VIP 时间，写 VIP 时间前后快照。
- `prop`：更新用户道具数量，写道具流水。
- medal 类奖励：不改余额，只写获得记录。

这个抽象带来的最大收益是：以后新增一个活动，不需要重新写“怎么加积分、怎么发道具、怎么写流水”。新的业务只需要回答三个问题：

1. 这次发奖的 `bizType` 是什么？
2. 这次发奖的 `bizId` 是什么？
3. 这次发哪些 `TaskRewardItem`？

回答完这三个问题，就可以复用统一发奖链路。

## 资产消耗服务：扣积分和扣道具也必须统一

和发奖相反，`UserAssetConsumeService` 负责资产消耗。它提供三个入口：

```java
ConsumeAssetResult consume(ConsumeAssetRequest request)

ConsumeAssetResult consumePoints(Long userId, Integer quantity, String bizType, Long bizId)

ConsumeAssetResult consumeProp(Long userId, Long propId, Integer quantity, String bizType, Long bizId)
```

积分消耗会检查用户积分余额，扣减后写一条 `change_type = sub` 的流水。道具消耗会 `selectForUpdate` 锁定用户道具行，检查数量是否足够，扣减后同样写流水。

统一消耗服务的意义和统一发奖服务一样：消耗动作本身是资产系统的核心能力，不应该散落在抽奖、道具使用、兑换等各个接口里。

比如抽奖优先消耗抽奖券，没有抽奖券再扣 100 积分。这个判断属于抽奖业务，但“扣抽奖券”和“扣积分”不属于抽奖业务，应该交给消耗服务。

## 抽奖服务：编排消耗、随机和发奖

`LotteryService` 的职责是编排，而不是亲自改资产。

一次抽奖大概分为几步：

1. 校验 `requestNo`。
2. 先查 Redis 结果缓存，如果相同请求已经成功过，直接返回历史结果。
3. 用 `userId + requestNo` 加 Redis 短锁，避免同一个请求号并发执行。
4. 判断优先使用抽奖券还是积分。
5. 调用 `UserAssetConsumeService` 扣道具或扣积分。
6. 按权重随机奖品。
7. 把奖品转换成 `TaskRewardItem`。
8. 调用 `ActivityRewardGrantService.grantGenericReward` 发奖。
9. 事务提交后缓存抽奖结果，方便客户端重试时拿到同样结果。

奖品配置目前是代码里的权重定义：

```text
10~30 积分：权重 55
50~70 积分：权重 15
80 积分：权重 5
下载券：权重 15
2小时 VIP 道具卡：权重 9
1天 VIP 道具卡：权重 1
```

抽奖随机使用 `ThreadLocalRandom`。随机积分奖品不是固定值，而是在配置的最小值和最大值之间再随机一次。比如抽中“10~30积分”后，实际发放值会在 10 到 30 之间生成。

这里有一个设计点：抽奖的 `bizId` 使用 `requestNo`。这样扣资产流水和发奖流水都能通过 `biz_type = lottery`、`biz_id = requestNo` 串起来。排查某次抽奖时，不需要额外的抽奖记录表也能看到这次请求扣了什么、发了什么。

## 道具使用：消耗和效果分开

道具使用比普通扣道具多一步：扣完道具之后，还要应用道具效果。

`PropUseService` 做的事情是：

1. 用 `requestNo` 做幂等。
2. 加 Redis 短锁。
3. 调用 `UserAssetConsumeService.consumeProp` 扣道具。
4. 如果道具是 VIP 卡，调用 `VipCardHandler` 延长 VIP。
5. 返回道具余额、VIP 到期时间、消耗流水 ID、效果流水 ID。

这里“消耗”和“效果”是两个概念。扣掉一张 VIP 卡是资产消耗，延长 VIP 时间是效果应用。两者都需要有记录，所以结果里会区分：

- `consumeLedgerId`：扣道具流水。
- `effectLedgerId`：应用 VIP 效果产生的流水。

如果中间失败，代码会把异常阶段、请求参数、错误信息记录到 Redis，并保留 30 天，方便后续排查。这个设计比只打一行错误日志更可靠，因为线上问题经常不是立刻发现的，日志可能已经滚动，但异常数据还可以按业务 key 找回来。

## 幂等设计：不要只相信接口层判断

活动系统最怕重复发奖和重复扣资产。前端重试、网络超时、用户连点、网关重放、服务端并发都可能导致同一业务被执行多次。

这套代码里用了几种不同层次的幂等手段。

### 1. 数据库唯一索引兜底

签到依赖：

```text
user_id + checkin_date
```

任务领取依赖：

```text
user_id + task_id + task_date
```

用户道具持仓依赖：

```text
user_id + prop_id
```

邀请绑定也类似，老 SQL 中可以看到 `inviter_id + invitee_id` 的唯一约束。

数据库唯一索引是最终防线。即使 Redis 锁失效、接口重复提交、应用多实例并发，只要写入同一条业务记录，数据库就能挡住重复插入。

### 2. Redis 短锁降低并发冲突

抽奖和道具使用都使用了 `userId + requestNo` 的 Redis lock key。

这个锁不是为了替代数据库事务，而是为了让同一个请求号不要并发执行两次。它更像是用户体验层和应用层的保护：如果第二个请求马上进来，要么拿到已缓存结果，要么得到“处理中”的提示，而不是打到数据库里争抢。

### 3. 结果缓存支持客户端重试

抽奖和道具使用成功后，会把结果按 `userId + requestNo` 缓存几天。客户端如果因为网络超时重试，同一个 `requestNo` 可以拿到相同结果。

这点对抽奖尤其重要。如果没有结果缓存，用户第一次请求其实已经抽中了奖励，但响应丢了；第二次请求又重新随机一次，就会造成体验和资产记录都不一致。

### 4. 行锁保证余额更新正确

道具扣减和道具发放使用 `selectForUpdate` 锁定 `vt_user_prop` 的具体行。这样同一个用户同一个道具的并发增减会串行化，避免余额覆盖。

积分更新目前是读取用户积分后计算 `afterPoints`，再调用更新方法写回。这里也要注意并发风险：如果未来积分消耗和发放并发更高，最好进一步升级为数据库原子更新，例如：

```sql
update vt_pockly_user
set points = points - #{quantity}
where id = #{userId}
  and points >= #{quantity}
```

然后通过影响行数判断余额是否足够。这样可以减少“读余额、算余额、写余额”之间的并发窗口。

## 事务边界：业务记录、资产变更、流水必须同生共死

发奖服务和消耗服务都使用了事务。原因很简单：业务记录、资产余额、流水记录必须保持一致。

任务领取时，不能出现 `vt_user_task` 插入成功但积分没加的情况；抽奖时，也不能出现积分扣了但奖励没发的情况；道具使用时，不能出现道具扣了但 VIP 效果没应用又没有异常记录的情况。

所以关键服务方法都用事务包起来：

- `grantTaskReward`
- `grantGenericReward`
- `consume`
- `consumePoints`
- `consumeProp`
- `lotteryDraw`
- `useProp`

另外，抽奖和道具使用的结果缓存要注意事务提交时机。代码里不是在业务刚执行完就盲目缓存，而是注册事务同步，在事务提交后再缓存结果。如果事务最后回滚了，却提前把成功结果写进 Redis，会导致客户端拿到一个数据库里并不存在的成功状态。

## 日志和可观测性

这次活动模块里日志打得比较细，每个关键接口都会记录：

- `userId`
- `taskId`
- `requestNo`
- `bizType`
- `bizId`
- 奖励摘要
- 流水 ID
- 余额变化
- 耗时

比如抽奖成功日志会包含扣了什么资产、消耗流水 ID、发奖流水 ID、奖励摘要。任务发奖日志会包含任务 ID、用户任务记录 ID、奖励列表、最终积分和 VIP 时间。

这些日志对活动系统特别重要。活动问题通常不是“接口报错”这么简单，而是用户会问：“我刚才抽奖为什么没到账？”、“我签到奖励是不是少了？”、“我用了 VIP 卡为什么时间没加？” 如果日志和流水设计完整，排查路径会短很多。

## AI 参与架构优化的收获

这次开发中，我觉得 AI 最大的价值不是帮我写某一段 CRUD，而是帮我不断追问架构边界。

一开始我会倾向于问：“帮我写一个签到接口”“帮我写一个抽奖接口”。如果按这个方向继续，很容易得到很多能跑但重复的代码。后来沟通重点变成：

- 签到、任务、邀请、抽奖的共同点是什么？
- 哪些逻辑应该属于业务编排，哪些应该属于资产服务？
- 奖励能不能抽象成统一模型？
- 消耗能不能抽象成统一模型？
- 重复发奖用什么兜底？
- 失败后怎么排查？
- 以后新增活动时，哪些代码不应该再改？

当问题从“写接口”变成“找稳定边界”后，架构自然就从接口 CRUD 走向了服务复用。

最后形成的结构是：

```text
业务接口负责表达场景
活动服务负责编排流程
发奖服务负责所有正向资产变化
消耗服务负责所有负向资产变化
流水表负责审计和报表
唯一索引、Redis 锁、结果缓存共同保证幂等
```

这个结构不是最复杂的，也不是为了设计而设计。它解决的是活动系统里最实际的问题：奖励类型会变，活动玩法会变，但资产发放和资产消耗必须稳定、统一、可追踪。

## 后续可以继续优化的点

现在这套架构已经比最初的 CRUD 模式清晰很多，但后续还有几个方向可以继续演进。

第一，奖品配置可以从代码迁移到数据库或配置中心。当前抽奖权重写在代码里，发布成本比较高。如果运营需要频繁调整概率，应该做成可配置。

第二，积分更新可以进一步原子化。尤其是抽奖、任务、签到都上线后，积分并发变化会更多，原子 SQL 或乐观锁版本号会更稳。

第三，流水表可以增加更严格的业务唯一键。例如对部分业务可以增加 `user_id + biz_type + biz_id + reward_type + change_type` 的唯一约束，进一步防止同一业务重复写流水。

第四，异常补偿可以产品化。现在道具使用失败会记录 Redis 异常数据，后续可以做一个后台页面，支持查看、筛选、人工补偿或标记处理。

第五，活动配置可以统一模型化。签到、任务、邀请、抽奖目前已经共享发奖能力，未来还可以继续抽象活动配置、参与条件、奖励规则、风控条件，让活动系统从“接口集合”进一步演进为“活动平台”。

这次重构最重要的经验是：活动系统不要被接口表象骗了。签到、任务、抽奖看起来是三个功能，本质上都是围绕用户资产变化展开的业务。只要抓住“发奖”和“消耗”这两个核心动作，再用流水、唯一索引、事务和缓存把边界守住，整个系统就会从一堆分散接口，变成一个可以继续生长的活动架构。
