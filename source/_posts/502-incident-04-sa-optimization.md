---
title: SA 风控接口从平均 2 秒到 100ms：一次只增不减的链路瘦身
date: 2026-07-14 20:30:00
tags:
  - Spring Boot
  - 性能优化
  - Redis
  - 风控
categories:
  - 后端优化
---

Prometheus 和 Grafana 上线后，SA 风控接口成了最明显的慢接口：平均耗时超过 2 秒，P95 波动很大，极端时达到 90 秒。初步优化完成后，平均耗时降到 100ms 以下，P95 稳定在 300ms 左右。

这次优化没有依赖某个神奇参数，而是清理了一条长期“只增不减”的调用链：废弃表仍在写、同一个 IP 被不同工具重复解析、缓存命中后仍查库、风控历史为了判断是否变化再查一次数据库、老用户在解析和判定结束时各更新一次。

相关提交集中在 7 月 9 日到 13 日：

```text
b96172c2  移除 SA 中保存 device_user_info 的逻辑
dcfbdac5  ipinfo 有结果时不再调用 ipUtils
b20c8f98  合并 PR/模拟器判断并清理重复解析
99e13501  1400+ 版本不再查询旧邀请奖励表
cebbfdb2  优化缓存命中与风控历史写入判断
f74d0642  老用户只在风控结束时更新一次
8548ef2b  风控配置改为 JVM 本地原子快照
```

## 一、先删除一张早已失去定位的表写入

`device_user_info` 在接手项目后已经不再作为核心数据源，但 SA 每次请求仍会执行设备初始化：先按 UUID 或 GAID 查询，找不到则插入，找到则更新设备数据。

也就是说，一张业务上已经废弃的表仍然为每个 SA 请求贡献至少一次 SQL，并可能产生锁等待。此前提交 `da0eb4e9` 甚至专门处理过初始化设备信息时锁等待超时导致的 SA 报错，这本身就是一个信号。

提交 `b96172c2` 删除了 V2/V3 风控服务对 `DeviceUserInfoMapper` 的依赖，同时移除：

```text
initOrUpdateDevice
insertDeviceUserInfo
updateDeviceUserInfo
allow / deny 中对 DeviceUserInfo 的回写
```

用户状态、设备原始数据和风控历史统一以 `PocklyUser` 为主，不再维护一份没有实际消费者的镜像数据。这个改动一次删除了 170 行左右的冗余逻辑，也直接减少了 SA 热路径上的数据库操作。

性能优化里，删除无效 I/O 往往比给旧代码加缓存更可靠。

## 二、不要让两个 IP 服务在主链路上重复工作

SA 会解析客户端上报的 IP 列表。原逻辑同时调用新版 ipinfo 和旧版 `ipUtils`，即使 ipinfo 已经返回有效结果，也会再次解析所有 IP；主 IP 还可能单独解析一次。

提交 `dcfbdac5` 把流程改为明确的主备关系：

```java
List<IPResponsePlus> ipInfoResults = ipList.stream()
        .map(ipInfoCoreUtils::getIpPlusResult)
        .filter(this::hasValidCountry)
        .collect(toList());

if (!ipInfoResults.isEmpty()) {
    // 使用 ipinfo 结果完成国家、VPN、hosting、ISP 等判断
} else {
    // 只有 ipinfo 无有效结果时，才调用 ipUtils 兜底
}
```

随后 `b20c8f98` 又复用 IP 列表第一个元素作为主 IP 结果，取消一次额外解析。

优化后的语义是：ipinfo 是主链路，`ipUtils` 是降级链路，而不是两个服务都跑一遍。对于新用户，Redis 没有 ipinfo 缓存时仍必须调用外部服务，实测高延迟时会超过 300ms；这也是当前 P95 很难继续压到几十毫秒的主要原因之一。

## 三、把便宜且确定的拒绝条件提前

风控链路里既有纯内存字段判断，也有数据库、Redis 和外部 HTTP 调用。执行顺序会直接影响平均成本。

`b20c8f98` 将 PR 为空判断提前，并把多处分散的 ABI、PR 完整性、模拟器特征和非法字段检查合并到 `checkPr`：

```java
private DeviceCheckResult checkPr(JSONObject pr, int sdk) {
    String abi = PrAbnormalDetector.resolveAbi(pr);
    String err = PrAbnormalDetector.detectAll(pr, abi);
    // aj/ag、an/al、模拟器特征、ABI 合法性等判断
    return new DeviceCheckResult(true, "ok");
}
```

ABI 统一从 `as`、`at`、`av` 三个候选字段中解析，避免 V2、V3 各自维护一套分支。能够通过请求字段立即判定的异常设备，不必继续访问数据库或第三方 IP 服务。

这类调整的原则很简单：不改变风控结果的前提下，把低成本、高淘汰率、无副作用的判断放在前面。

## 四、缓存命中就直接返回，不要又走一遍写路径

SA 已经缓存了用户最近一次风控结果。旧逻辑命中缓存后仍会调用 `allowUser` 或 `deny`，拒绝分支还会再次查用户 remark。这会继续触发用户更新、结果缓存刷新和历史记录判断，缓存只省掉了一部分检测，却没有真正把热路径截断。

提交 `cebbfdb2` 改为：

```java
if ("1".equals(cached)) {
    return new RiskResponse<>(RISK_VERSION, buildUserVo(user, true));
}
return new RiskResponse<>(RISK_VERSION, buildUserVo(user, false));
```

缓存命中后直接构建响应，不再为了得到同一个风控结论额外查询用户 remark，也不会重复更新用户、刷新判定结果或写风控历史。`buildUserVo` 中仍有权益等响应数据需要查询，这部分在后续按版本继续裁剪。

## 五、判断“状态是否变化”不需要再查一次历史表

旧版 `recordIfChanged` 会查询用户最新一条风控历史，再与当前 `userType`、`remark` 比较。实际上进入 allow/deny 前，内存中的用户对象已经带有旧状态。

优化后先保存变更前的值：

```java
Integer previousUserType = user.getUserType();
String previousRemark = user.getRemark();

user.setUserType(1);
user.setRemark("");

recordIfChanged(user, previousUserType, previousRemark);
```

然后直接比较前后状态，只有变化时才插入风控历史：

```java
boolean typeChanged = !Objects.equals(
        previousUserType, user.getUserType());
boolean remarkChanged = !StrUtil.equals(
        previousRemark, user.getRemark());
```

这让每次完整风控至少少一次“查询最新历史”的 SQL，同时仍保留状态变化审计。

## 六、老用户只更新一次，旧版本逻辑只服务旧版本

`resolveUserByDevice` 原本在解析老用户时就执行一次 `updatePocklyUser`，风控结束的 allow/deny 又会更新一次。提交 `f74d0642` 删除了解析阶段的更新，统一到风控结束时落库。

另一个长期成本来自兼容逻辑。1400 以上版本已经合并 VIP 权益，却仍查询旧邀请奖励表，而且先查询未领取数量，`confirmClaim` 内部又查一次。提交 `99e13501` 做了两件事：

- 1400 及以上版本完全跳过旧邀请奖励表。
- 旧版本直接使用 `confirmClaim` 的返回值，不再提前重复查询。

兼容代码最危险的地方不是复杂，而是它常常永远不会被删除。给版本分支设定清晰边界，能避免所有新用户一直为历史逻辑付费。

## 七、把每次请求读 Redis 改成读 JVM 快照

风控入口还需要读取 GAID 白名单、忠诚用户白名单、审核开关和审核版本。此前这些配置会在请求中访问 Redis。

提交 `8548ef2b` 新增 `RiskConfigLocalCacheServiceImpl`，启动时从 Redis 预热，所有请求只读取 `AtomicReference` 中的不可变快照：

```java
private final AtomicReference<RiskConfigSnapshot> snapshot =
        new AtomicReference<>(RiskConfigSnapshot.empty());
```

刷新时先完整读取四项配置，再一次性替换：

```java
snapshot.set(new RiskConfigSnapshot(
        immutableSet(whitelist),
        immutableSet(loyalWhitelist),
        Boolean.parseBoolean(auditSwitch),
        appVersion));
```

这样并发请求只可能看到完整旧快照或完整新快照，不会读到刷新到一半的混合状态。多实例同步通过 Redis Pub/Sub channel 完成：

```text
risk:local-cache:refresh
```

收到通知后，各实例重新从自己连接的 Redis DB 加载配置。刷新失败则保留旧快照并发送飞书告警，避免临时 Redis 异常把白名单清空。刷新成功也会携带主机名、IP、审核开关和版本发送消息，便于发布前确认两台服务器状态一致。

这不是为了把所有 Redis 数据都塞进本地缓存，而是因为这几项配置数据量小、读频率高、变更频率低，非常适合快照化。

## 八、结果与仍然存在的上限

完成第一轮优化后，监控数据从：

```text
平均耗时 2s+
P95 峰值约 90s
```

降到：

```text
平均耗时 < 100ms
P95 稳定在约 300ms
```

当前尾延迟主要受新用户 ipinfo 缓存未命中影响。后续如果继续优化，优先级应该是：

1. 给 ipinfo 单独增加超时、耗时直方图和失败率监控。
2. 确认调用外部服务时没有持有数据库事务或连接。
3. 对可接受短期旧数据的 IP 结果增加多级缓存。
4. 用 trace 串起 SA、SQL、Redis 和 ipinfo，定位剩余长尾。

## 九、为什么没有立即把 FCM 拆成微服务

事故后提出把 FCM 拆成独立服务是合理方向，它能隔离异步消费与 API 的资源。但这次指标已经表明，SA 本身存在大量可删除的数据库和外部调用。只拆 FCM 不会自动让 SA 从 2 秒变成 100ms。

因此我更倾向于先完成可观测性和热路径优化，再进行服务拆分。中间如果容量仍不足，可以先通过 Spring Boot profile/role 启动 FCM 专属实例，实现部署层隔离；等边界、配置、监控和失败处理稳定后，再建立独立项目，顺便舍弃老旧代码。

服务拆分是架构演进，不应该成为事故中替代根因分析的快捷答案。

## 总结

SA 的性能问题不是一个慢 SQL 或一个 Redis 调用造成的，而是多年累积的小成本叠加：

```text
废弃表写入
+ 重复 IP 解析
+ 缓存命中后的额外查写
+ 风控历史查询
+ 老用户重复更新
+ 旧版本奖励逻辑
+ 高频 Redis 配置读取
```

这次优化的核心方法也很朴素：先用 P95 找到最慢入口，再沿调用链删除无效 I/O、合并重复操作、提前便宜判断、让缓存真正截断链路。最终平均耗时从秒级回到百毫秒以内，比单纯扩大线程池或数据库连接池更接近问题本质。
