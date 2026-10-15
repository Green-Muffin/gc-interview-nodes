# Unreal Engine 面试题与答案

> 共 6 题。按主题分类，答案直接放在题目后。

[返回题库总目录](面试题库总目录.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、Gameplay 与对象生命周期](#%E4%B8%80%E3%80%81Gameplay%20%E4%B8%8E%E5%AF%B9%E8%B1%A1%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F) | 1–4 |
| [二、网络](#%E4%BA%8C%E3%80%81%E7%BD%91%E7%BB%9C) | 5–6 |

## 一、Gameplay 与对象生命周期

### 1. GAS 的核心组成和执行流程是什么？

**答案：**

Ability System Component（ASC）管理角色拥有的 Ability、Attribute、Gameplay Effect、Gameplay Tag 和 Gameplay Cue。AttributeSet 保存生命、能量等属性；Gameplay Effect 描述瞬时或持续修改；Ability 组织激活、任务、消耗和结束流程。

典型流程是授予 Ability → 检查 CanActivate、标签、消耗和冷却 → ActivateAbility → AbilityTask 等异步步骤 → 应用 Effect／Cue → EndAbility 或 Cancel。多人游戏中通常由服务端权威执行，Local Predicted Ability 可先在客户端响应，再由服务端确认或回滚可预测部分。

### 2. UE 的 UObject GC 如何实现？有什么缺点？

**答案：**

UObject GC 以根集合和被引擎跟踪的强引用为起点做可达性分析，标记不可达对象并清扫。UPROPERTY／TObjectPtr、容器和 AddReferencedObjects 等机制让引用进入追踪图；原生非 UObject 资源仍需 RAII 或显式生命周期管理。

缺点是可达性扫描和清理会消耗时间，大量 UObject 可能产生帧尖峰；引用标记错误还可能导致对象被提前回收或长期保留。控制对象数量、分批创建／销毁、正确使用弱引用和增量可达性能力可降低影响。

### 3. UE 中对象池如何实现？有哪些缺点？

**答案：**

预先创建一批 Actor／UObject，空闲时隐藏、停 Tick、停碰撞并放回可用队列；获取时恢复位置和状态，归还时清除事件、计时器、网络状态和外部引用。池满时按策略销毁或拒绝归还。

对象池减少频繁 Spawn／Destroy 和 GC 压力，但会提高常驻内存，重置不完整会泄漏上次状态。对重量差异大、使用频率低的对象，池可能不划算；应根据峰值和命中率限制容量。

### 4. UE 反射系统如何生成和使用类型信息？

**答案：**

UCLASS、UPROPERTY、UFUNCTION 等宏标记要暴露给 UE 的类型与成员。构建流程中的 Header Tool 读取这些声明并生成注册／包装代码，运行时形成 UClass、FProperty、UFunction 等元数据。

序列化、GC 引用追踪、编辑器详情面板、Blueprint 调用和 RPC 都依赖这些元数据。普通未标记 C++ 成员不自动进入 UE 反射；修改属性名称／类型时还要考虑资源序列化兼容。

## 二、网络

### 5. UE 网络同步的核心模型是什么？

**答案：**

UE 通常采用服务端权威模型。NetDriver／NetConnection 维护连接，Actor、Component 和属性按相关性、频率及条件复制；RPC 发送事件调用，RepNotify 处理属性变化后的逻辑。

只复制需要的状态，客户端用预测和校正改善响应。Actor 必须满足复制与所有权条件，RPC 方向也受 owning connection 限制；不能把每帧所有对象完整发送。

### 6. UE 底层使用 UDP 时，可靠消息如何处理丢包？

**答案：**

UDP 只提供数据报。UE 的网络层在其上维护包序号、确认信息、连接／Channel 状态，并对可靠 bunch／RPC 进行排序和重传；确认包本身丢失时，发送方因为未收到确认仍会继续重传，直到确认或连接超时。

可靠消息过多会占用窗口并阻塞同 Channel 后续可靠数据。高频可丢状态应使用属性复制或不可靠 RPC，让新状态替代旧状态；带宽、重传和拥塞需要一起控制。

## 参考资料

- [Epic：Gameplay Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-the-unreal-engine-gameplay-ability-system)
- [Epic：Incremental Garbage Collection](https://dev.epicgames.com/documentation/unreal-engine/incremental-garbage-collection-in-unreal-engine)
- [Epic：Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine)
