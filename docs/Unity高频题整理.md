# Unity 客户端高频面试题与答案

> Unity 生命周期、资源、渲染实践、UI、物理、动画与工程设计。历史题按标题标注；引擎与插件行为应结合项目版本。 共 86 题。

本专题以概念、机制、边界和实战追问组织；示例可能省略外围业务代码。

[返回题库总目录](面试题库总目录.md)

## 目录

| 主题 | 题号 |
| --- | --- |
| [一、对象、生命周期与协程](#一、对象、生命周期与协程) | 1–14 |
| [二、物理与 UGUI 基础](#二、物理与%20UGUI%20基础) | 15–28 |
| [三、渲染管线与图形优化](#三、渲染管线与图形优化) | 29–45 |
| [四、资源、依赖与内容更新](#四、资源、依赖与内容更新) | 46–58 |
| [五、性能、GC 与对象池](#五、性能、GC%20与对象池) | 59–68 |
| [六、异步、Job System 与数据并行](#六、异步、Job%20System%20与数据并行) | 69–74 |
| [七、架构与系统设计](#七、架构与系统设计) | 75–78 |
| [八、脚本后端与原生交互](#八、脚本后端与原生交互) | 79–82 |
| [九、动画、平台与旧项目维护](#九、动画、平台与旧项目维护) | 83–86 |

## 一、对象、生命周期与协程

### 1. `Awake`、`OnEnable`、`Start`、`FixedUpdate`、`Update`、`LateUpdate` 如何执行？

**答案：**

可以先描述同一组件的初始化，再描述不同更新阶段，不能把它们背成每帧固定执行一次的直线序列。

| 回调 | 时机与用途 |
| --- | --- |
| `Awake` | 实例初始化时调用一次；常用于本对象内部状态和组件引用初始化 |
| `OnEnable` | 对象在层级中激活且组件启用时调用；可重复发生，适合绑定启用期订阅 |
| `Start` | 脚本首次满足启用条件后，在首次更新前调用一次 |
| `FixedUpdate` | 按固定模拟步长调度，用于物理相关逻辑；一个渲染帧可执行零次、一次或多次 |
| `Update` | 启用状态下每帧更新，适合输入采集和普通逻辑 |
| `LateUpdate` | 在 Update 阶段之后更新，常用于相机跟随和后处理逻辑 |

**边界：**处于激活对象上的禁用脚本也可能已执行 `Awake`；初始不激活的对象通常要等激活才初始化。固定步长不意味着在墙钟时间上总能准时执行；负载过高时还受最大时间步限制。不同对象的同名回调没有任意可依赖的顺序，运行时创建对象也不能套用“所有对象都已 Awake”的假设。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/execution-order.html)；[官方资料 2](https://docs.unity3d.com/ja/6000.0/ScriptReference/MonoBehaviour.Awake.html)。

### 2. `SetActive(false)` 与 `enabled = false` 有什么区别？对协程和 `Invoke` 有什么影响？

**答案：**

`SetActive` 修改 GameObject 的本地激活状态；最终是否生效还要看父对象，即 `activeInHierarchy`。`enabled` 只控制相应 Behaviour 组件，不等于关闭整个物体。

| 操作 | 主要影响 |
| --- | --- |
| `gameObject.SetActive(false)` | 对象及相关子层级失活，渲染、物理和脚本更新等受影响；承载于其上的协程停止 |
| `behaviour.enabled = false` | 停止该脚本的常规更新回调；已启动协程不会仅因此停止 |
| `Destroy(behaviour)` | 销毁组件，组件承载的协程会停止 |

不能把 `Invoke`／`InvokeRepeating` 当成 `Update` 或协程来推断。**失活、禁用不应作为取消预约的业务机制**，已预约的调用不能据此假定已清除；需要停止时显式 `CancelInvoke()`。如果业务要求“隐藏即暂停”，应在 `OnDisable` 取消，在 `OnEnable` 按规则重新注册。重新激活也不会自动恢复已经停止的协程执行位置。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/Coroutines.html)；[官方资料 2](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/MonoBehaviour.CancelInvoke.html)。

### 3. 协程有什么作用和原理？与多线程、`async/await` 有什么区别？

**答案：**

Unity 协程通常由 C# 迭代器状态机和主线程调度器组成。执行到 `yield` 时交还控制权，在等待条件满足后的相应调度阶段继续执行。

- **协程**：适合等待帧、表现流程、延时和分帧处理；普通协程代码仍在主线程，长时间计算照样卡帧。
- **多线程**：允许计算并行，需要处理共享状态、同步和引擎 API 的线程限制。
- **`async/await`**：组织异步操作的控制流，可等待 I/O 或任务；不自动创建线程，也不天然绑定 MonoBehaviour 生命周期。

协程可以等待资源或场景异步操作，但不会把普通同步加载函数自动变成异步函数。虽然协程通常不并行执行，多个协程仍可能跨帧交错修改状态，因此仍需防止重复流程和时序冲突。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/Coroutines.html)。

### 4. 常见 `yield` 写法有什么区别？

**答案：**

| 写法 | 典型恢复条件 |
| --- | --- |
| `yield return null` | 下一帧继续 |
| `yield return new WaitForSeconds(1f)` | 按受 `timeScale` 影响的游戏时间等待；不是精确定时器 |
| `yield return new WaitForSecondsRealtime(1f)` | 按不受 `timeScale` 影响的时间等待 |
| `yield return new WaitForFixedUpdate()` | 等待后续固定更新阶段 |
| `yield return StartCoroutine(Other())` | 等待子协程完成 |
| `StartCoroutine(Other())` | 启动子流程，当前流程不因此等待它完成 |

`WaitForSeconds` 从当前帧结束后开始计时，到时后也要等可恢复的帧；暂停游戏时可能一直等不到。创建迭代器和等待对象可能有分配，但缓存策略要考虑等待对象是否包含可变状态，不能盲目复用所有 yield 对象。

### 5. 销毁对象后协程会怎样？如何防止协程持有过期引用？

**答案：**

销毁承载的 GameObject 或 MonoBehaviour，会停止其协程。仅设置脚本 `enabled = false` 不会停止它；GameObject 失活会停止。

在明确的生命周期节点保存并停止协程句柄，解除事件和资源引用。对于可能永久等待的流程，加入退出条件、超时或取消入口，不依赖协程正常执行到最后一行才清理。

```csharp
private Coroutine routine;

void OnEnable()
{
    routine = StartCoroutine(RefreshLoop());
}

void OnDisable()
{
    if (routine != null)
        StopCoroutine(routine);
    routine = null;
    CancelInvoke();
    // 在这里解除本次启用期持有的订阅和资源
}
```

`DontDestroyOnLoad` 用于让合适的根对象跨场景保留，不是对显式 `Destroy` 的免疫；长期存活的管理器协程更需要主动解除对旧场景的引用。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/Coroutines.html)。

### 6. Prefab、Prefab Variant 与嵌套 Prefab 分别解决什么问题？

**答案：**

- **Prefab**：保存 GameObject 层级、组件和序列化数据，作为可复用模板。
- **Variant**：在基础 Prefab 上保留差异覆盖，例如同类敌人的颜色、血量和装备。
- **嵌套 Prefab**：组合可独立维护的模块，例如窗口嵌套通用按钮和列表项。

编辑器中的实例和变体可继承资源修改，但本地覆盖项可能遮住变化；运行时克隆出的对象也不是会自动同步全部配置的远程“模板订阅者”。设计时应控制变体继承深度，明确哪些字段允许覆盖，避免到处 Apply／Revert 导致误修改。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/PrefabVariants.html)。

### 7. `Instantiate` 为什么昂贵？如何减少集中生成造成的卡顿？

**答案：**

可以从可观察的工作理解：克隆对象及子层级、创建组件和原生／托管相关对象、复制序列化数据、建立引擎关联，并在满足激活条件时触发生命周期回调。

真正的开销还可能来自 `Awake`、`OnEnable` 中的资源查询、订阅、布局和逻辑，不只是内存分配。非激活实例不能简单套用“立即执行所有回调”的流程。

优化可采用分帧预热、对象池、简化 Prefab 层级、减少初始化热点、延迟非必要组件，并分别测量加载、实例化、激活三个阶段。异步加载 Prefab 不代表实例化和激活一定没有主线程成本。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/Object.Instantiate.html)。

### 8. `GUID`、`fileID` 与 `InstanceID` 有什么区别？

**答案：**

| 标识 | 作用与范围 | 能否作为长期资源身份 |
| --- | --- | --- |
| `GUID` | 项目资源的标识，通常保存在 `.meta`；保留 meta 的正常移动或重命名可保持 GUID | 可以作为编辑期资源身份，但删除重建 meta 会改变 |
| `fileID`／Local ID | 标识某个资源文件中的具体对象或子资源 | 通常与 GUID 配合定位；不是跨所有文件唯一 |
| `InstanceID` | 当前会话中已加载 Unity 对象的临时身份 | 不应用作存档或跨运行会话标识 |

序列化引用常由 `GUID + fileID` 表达；不能说 fileID 都“位于同一个 .meta 里”。跨场景持久对象、共享资源不会因一次场景切换就全部消失。对象身份、资源身份和业务实体 ID 应分别设计。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/AssetDatabase.TryGetGUIDAndLocalFileIdentifier.html)。

InstanceID 不是应用可依赖的 `GUID + fileID` 哈希计算结果；运行时新建的对象也可能没有持久化资源 GUID。不要自行重建这一映射作为引擎保证。

### 9. Unity 对象的 `== null` 与普通 C# 引用判空有什么区别？

**答案：**

`UnityEngine.Object` 重载了 `==`，判空不仅涉及托管引用，也涉及对应原生对象是否仍存在。因此托管包装对象仍存在时，`obj == null` 也可能为真。

`ReferenceEquals(obj, null)` 检查引用身份；`is null` 不调用重载的 `==`。`?.` 和 `??` 也不使用 Unity 的自定义相等运算符，不能直接替代引擎对象有效性检查。

参考：[UnityEngine.Object 官方说明](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/Object.html)。

### 10. 为什么事件订阅可能造成内存泄漏？Unity 中如何避免？

**答案：**

发布者通过委托持有订阅者的引用。如果长生命周期的发布者持有短生命周期的订阅者，即使订阅者在业务上已经不再使用，也可能仍然可达，无法被 GC 回收。

当订阅的有效期与组件启用期一致时，可以成对订阅和取消订阅：

```csharp
void OnEnable()
{
    GameEvents.OnScoreChanged += HandleScoreChanged;
}

void OnDisable()
{
    GameEvents.OnScoreChanged -= HandleScoreChanged;
}
```

重点检查静态事件、全局事件总线和对象池中的重复订阅。订阅者销毁或不再需要事件时，应根据实际生命周期解除订阅。

### 11. Transform 的继承关系与本地／世界变换如何理解？

**答案：**

`Transform` 继承自 `Component`，记录位置、旋转、缩放以及父子层级关系。`position`／`rotation` 面向世界空间，`localPosition`／`localRotation`／`localScale` 相对父级。

`Transform.Rotate` 可通过 `Space.Self` 或 `Space.World` 明确旋转空间。设置父级时要明确是否保留世界姿态；父级非均匀缩放与旋转组合可能产生难以用单独缩放向量描述的效果。与刚体交互时优先遵守物理移动流程，而非任意改 Transform。

### 12. PlayerPrefs 如何保存整数？适合做什么？

**答案：**

```csharp
PlayerPrefs.SetInt("BestScore", bestScore);
int savedScore = PlayerPrefs.GetInt("BestScore", 0);
PlayerPrefs.Save(); // 在合适的保存节点调用
```

适合音量、偏好等少量本地设置。它不是安全数据库，不应用来保护付费资产、令牌等秘密；频繁 `Save()` 可能产生磁盘开销。复杂存档应考虑 schema 版本、原子写入、备份与损坏恢复。

### 13. 如何让对象跨场景保留？如何避免重复单例？

**答案：**

在合适的根 GameObject 上调用 `DontDestroyOnLoad(gameObject)`，通常连同其子层级一起保留。它不会阻止显式销毁，也不会自动更新旧场景对象引用。

推荐通过启动场景或明确的全局作用域管理服务；新场景若也包含同类管理器，要检测并销毁重复实例。全局对象要解除旧场景事件和资源引用，异步返回时检查场景版本。现代场景加载使用 `SceneManager` API，旧资料的 `LoadLevel` 属于历史写法。

### 14. 组件字段或脚本出现 Missing／数据丢失，应如何排查？

**答案：**

按引用与序列化链检查：对象是否被删除、资源与 `.meta` 是否一起迁移、脚本类型或程序集是否改名、字段名／类型是否变化、字段是否满足序列化规则，以及 Prefab 覆盖是否被还原。

普通字段重命名可在适用时使用 `FormerlySerializedAs` 保留旧数据；类型或程序集迁移需要相应迁移方案。先查看版本差异与错误日志，不要一上来删除 meta 或重挂所有脚本，这可能扩大引用损失。

## 二、物理与 UGUI 基础

### 15. `Collider`、`Rigidbody`、`Is Kinematic` 与触发事件如何配合？

**答案：**

以下讨论内置 **3D 物理**，不要与 `Rigidbody2D` 的规则混用。

`Collider` 定义碰撞形状；`Rigidbody` 定义刚体运动。没有刚体的 Collider 可以作为静态地面或墙壁，因此并不要求碰撞双方都有 Rigidbody。

- 普通碰撞回调：双方具有有效 Collider，均非 Trigger，通常至少一方关联非运动学 Rigidbody，并满足层碰撞矩阵等规则。
- Trigger 回调：至少一方勾选 `Is Trigger`，且通常至少一方关联 Rigidbody；用于检测重叠，不提供普通碰撞阻挡响应。
- `isKinematic = true`：由程序或动画控制运动，不受普通力和重力驱动；仍可按碰撞矩阵与其他对象交互，适合移动平台等。

移动运动学刚体时可在固定更新阶段使用合适的刚体移动 API。复合 Collider 的刚体可能在父级，不应只检查碰撞体所在节点。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/collider-types-interaction.html)。

接触过程常对应 `OnCollisionEnter/Stay/Exit`，触发重叠对应 `OnTriggerEnter/Stay/Exit`；对象销毁、失活等情况下不能把 Exit 回调当作唯一清理入口。Trigger 仍参与物理系统的重叠检测，不是被物理系统完全忽略。

### 16. 刚体插值与 CCD 有什么区别？高速子弹穿透如何处理？

**答案：**

**Interpolation** 用相邻物理状态平滑视觉表现，缓解物理步长与显示帧率不同造成的抖动；它不提高碰撞检测频率，也不直接解决穿透。

离散碰撞检测只检查离散时刻，物体一步跨过薄墙时可能漏检。常见方案：

1. 为需要的刚体选择合适的连续碰撞检测模式（CCD），注意支持的形状、运动和开销。
2. 对子弹使用上一位置到下一位置的 Raycast／SphereCast 等扫掠检测，处理碰撞层、触发器和最近命中。
3. 必要时缩小物理时间步长或对子弹运动细分，评估额外 CPU 成本。
4. 调整碰撞体厚度、限制极端速度，或对命中扫描武器采用 Hitscan。

直接每帧修改 Transform 绕过物理运动，也可能破坏预期的检测过程。CCD 不是对所有几何与所有运动都绝对不漏的保证。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/ContinuousCollisionDetection.html)。

### 17. UGUI 的 Anchor 与 Pivot 有什么区别？如何适配屏幕？

**答案：**

Anchor 是相对父 RectTransform 的归一化参考点或区域，决定定位和拉伸关系；Pivot 是自身矩形的归一化参考点，影响定位以及旋转、缩放中心。

常见适配流程：

1. 按语义设置锚点：角落按钮固定到相应边角，全屏背景采用拉伸锚点。
2. 在拉伸模式下设置边距，而不是单纯写死宽高和坐标。
3. Canvas Scaler 选择 `Scale With Screen Size`，设置参考分辨率与 Match 策略。
4. 处理安全区域、刘海、极端宽高比、文本换行和本地化。

Canvas Scaler 不能自动解决所有布局问题；自适应也不意味着在不同宽高比下简单等比例放大所有元素。

参考：[官方资料 1](https://docs.unity3d.com/Packages/com.unity.ugui@2.0/manual/UIBasicLayout.html)。

### 18. Canvas 三种渲染模式如何选择？拆 Canvas 和射线检测如何优化？

**答案：**

| 模式                     | 用途与特点                  |
| ---------------------- | ---------------------- |
| Screen Space - Overlay | 直接呈现在屏幕空间，不依赖用于 UI 的相机 |
| Screen Space - Camera  | 由指定相机呈现，需考虑相机、排序和裁剪设置  |
| World Space            | UI 作为世界中的面板，受视角、距离等影响  |

没有“某模式永远最快”的结论。Canvas 中部分内容频繁变化时，可按更新频率和层级拆分动态区与静态区，缩小重建范围；但过度拆分会增加 Canvas 和批次成本，也不能消除透明像素叠加造成的 Overdraw。

UGUI 通过 EventSystem、输入模块和 GraphicRaycaster 找出候选 Graphic，并结合排序、遮挡与过滤规则确定命中。关闭非交互图像的 `raycastTarget`，只在需要交互的 Canvas 放置 GraphicRaycaster，避免不必要的物理阻挡检测和复杂 Layout 重建。

参考：[官方资料 1](https://unity.com/how-to/unity-ui-optimization-tips)。

### 19. CharacterController 与 Rigidbody 有什么区别？

**答案：**

CharacterController 是面向角色移动的胶囊碰撞控制组件，不是“受限的 Rigidbody 子类”。`Move` 执行带碰撞约束的位移，通常需要业务代码自己处理重力、跳跃等；它不会自动像动态刚体一样响应外力。

Rigidbody 用于物理模拟，可受力、重力与碰撞冲量驱动。选择取决于需要可控的角色手感还是物理交互；不要让两套控制同时争夺同一个对象的位置。

### 20. AddForce 的四种 ForceMode 有什么区别？

**答案：**

对 3D 动态刚体，可从速度变化理解：

| ForceMode | 概念 | 典型速度变化 |
| --- | --- | --- |
| Force | 持续力，考虑质量 | `Δv = F × Δt / m` |
| Acceleration | 加速度，不考虑质量 | `Δv = a × Δt` |
| Impulse | 一次冲量，考虑质量 | `Δv = J / m` |
| VelocityChange | 直接速度变化 | `Δv = 输入值` |

`AddForceAtPosition` 还可能因作用点偏离质心产生力矩；转动可使用 `AddTorque`。连续施力通常在固定更新中安排，不要再盲目乘一次 `deltaTime`；公式未计入阻力、约束等后续模拟影响。

### 21. HingeJoint 是链条距离约束吗？

**答案：**

HingeJoint 是铰链关节，主要允许物体围绕指定轴旋转，适合门、摆臂和链条中的单个连接点，可设置弹簧、马达和角度限制。

“自由移动到固定距离才产生拉力”不是 HingeJoint 的通用定义。距离、弹性或更复杂的自由度限制应根据需求选择 SpringJoint、ConfigurableJoint 等。链条通常由多个刚体和关节组成，还需控制质量比、迭代精度与性能。

### 22. Physics.Raycast 是真的发射物体吗？如何选择查询 API？

**答案：**

Raycast 是沿原点、方向和长度进行空间查询，不会创建一个飞行实体。`Raycast` 通常取最近命中，`RaycastAll` 取得多个命中但不保证排序；NonAlloc 版本可复用结果缓冲，要处理容量不足。

正确设置 `maxDistance`、LayerMask 和 Trigger 查询策略。屏幕点击可先由相机生成射线；有体积的运动检测可能需要 SphereCast 或 CapsuleCast。射线与线段／AABB 的数学原理见 [图形学与3D数学题库](图形学与3D数学题库.md)。

### 23. Unity 如何做 2D？IMGUI、UGUI 与 UI Toolkit 如何分工？

**答案：**

2D 游戏可使用 SpriteRenderer、Tilemap、2D 物理、动画及合适的相机投影，并不需要把所有内容做成 UI，也不必依赖老插件。

IMGUI 使用 `OnGUI` 等即时模式，适合部分调试和编辑器工具；回调可能因不同事件在一帧多次执行。UGUI 使用 Canvas、组件和事件系统，适合常见运行时 UI。UI Toolkit 使用视觉树、样式和布局系统，具体运行时能力需结合 Unity 版本。按需求选型，不用“某套 UI 全面取代所有其他方案”概括。

### 24. 动态字体与静态字体图集如何选择？

**答案：**

静态字形图集预先准备固定字符，运行时较可控，但多语言字符集很大时会增加图集与维护成本。动态图集按需加入字形，适合不易提前枚举的文本，但可能产生首次字形准备、图集增长和回退字体开销。

动态字体不天然在所有 Unicode 场景都更好。规划常用字预热、fallback、图集容量、字体授权与文本刷新频率，避免聊天文本或语言切换引发集中开销。

### 25. UGUI 从数据变化到渲染大致经历什么？为什么同材质仍不能合批？

**答案：**

属性变化可能将布局、顶点或材质标为 dirty；随后在相应更新阶段重建布局与几何，Canvas 汇集可绘制内容，并按材质、纹理、裁剪及顺序等约束生成批次。

同材质、同贴图只是条件之一。Canvas 边界、Mask／RectMask、层级穿插、透明排序和渲染状态都可能打断合批。性能分析要分别看布局重建、网格更新、批次和透明 Overdraw，不能把它们都归结为 Draw Call。

### 26. UI 之间的粒子特效如何控制前后层级？

**答案：**

先确定 UI 的 Canvas 模式和粒子的渲染路径。普通 ParticleSystem Renderer 属于场景渲染，不能仅靠提高 Sorting Order 就任意插入 Screen Space - Overlay Canvas 的两个 Graphic 之间。

常见方案是改用合适的 Camera／World Space Canvas 配合排序，使用能接入 Canvas 绘制的 UI 粒子方案，或把特效渲染到 RenderTexture 再用 RawImage 显示。分别评估遮罩、透明、后处理、分辨率和额外相机成本；跨相机时还要检查目标与清除设置。

### 27. 背包有几万条数据，UI 列表如何展示？

**答案：**

使用虚拟化列表：完整数据与少量可见格子分离，只创建视口范围加少量缓冲的展示实例。滚动时计算数据索引并复用格子，避免为全部道具创建 GameObject。

固定高度列表可直接由滚动偏移计算首个索引；可变高度需要高度缓存与前缀索引。每次绑定设置版本号，异步图标回调校验当前道具，回收时解绑事件并归还资源。批量更新差量数据，减少 Layout 重算和每帧字符串分配。

### 28. 【历史题】NGUI 的 Grid、Table、Panel 与 Anchor 分别做什么？

**答案：**

在维护 NGUI 老项目时，UIGrid 常用于规则网格／行列布局，UITable 更适合按子控件尺寸组织表格；UIPanel 负责组织 Widget、裁剪与相关绘制批次；UIAnchor 用于相对屏幕或参考对象对齐。

这些是特定插件的类型，不等同于 UGUI 的同名概念。旧资料中的 half-pixel 修正依赖历史图形 API 与实现，不应当作现代跨平台布局的通用步骤；具体属性以项目使用的 NGUI 版本为准。

## 三、渲染管线与图形优化

### 29. Material 与 Shader 有什么关系？如何实现纯色或自发光？

**答案：**

Shader 定义 GPU 使用的着色程序与相关渲染配置；Material 引用 Shader，并保存纹理、颜色、数值、关键字等参数。多个材质可以使用同一 Shader。

Standard Shader 是 **Built-in 管线**的常见 PBR Shader；URP／HDRP 应选择对应的 Lit 或 Unlit，不是直接通用替换。

- 只需要不受光照影响的纯色：选择对应管线的 Unlit 材质，设置颜色并赋给 Renderer。
- 需要发光表面：使用支持 Emission 的材质并设置 HDR 发光颜色与强度。
- 需要视觉光晕：还需合适的曝光和 Bloom；只勾选 Emission 不保证出现光晕，也不等于一定照亮周围物体。

发光参与烘焙或实时 GI 的方式取决于管线和配置。

### 30. BIRP、URP 与 HDRP 有何核心差异？URP 为什么常用于移动端？

**答案：**

| 管线 | 定位 | 主要取舍 |
| --- | --- | --- |
| Built-in／BIRP | 传统内置流程，历史项目与资源生态较多 | 定制入口较分散，不能像 SRP 一样整体定义渲染循环 |
| URP | 面向较广硬件范围的可配置 SRP | 便于控制灯光、阴影、分辨率与特效预算，常用于移动端与跨平台项目 |
| HDRP | 面向高端硬件的高保真渲染 | 高级光照与画面能力更多，硬件、带宽和资源成本更高 |

URP 的价值在于可缩放的功能与预算控制，不是“名字里有 Universal 就一定更快”。过多全屏 Pass、实时阴影、透明叠加和过高分辨率仍会拖慢移动端。

三者是选型关系，不是 BIRP → URP → HDRP 的必然升级路径。迁移需检查 Shader、材质、后处理与插件兼容性。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/render-pipelines-overview.html)。

### 31. SRP 给开发者什么控制权？为什么影响性能与画面？

**答案：**

SRP 通过 C# API 允许开发者组织渲染命令：决定相机如何执行剔除、筛选和排序对象、安排绘制 Pass、设置渲染目标与处理后期效果。URP 和 HDRP 是基于 SRP 的具体实现。

它能改变光照方案、深度预通道、阴影、后处理和中间纹理生命周期，因此同时影响画面、CPU 提交成本、GPU 工作量与显存带宽。

开放的是渲染流程的组织能力，不是任意改写 GPU 硬件或所有引擎内部逻辑。定制 Pass 也有维护、资源读写和同步成本；新旧 URP 的 Renderer Feature／RenderGraph 扩展方式应按版本选用。

参考：[官方资料 1](https://docs.unity3d.com/es/2021.1/Manual/scriptable-render-pipeline-introduction.html)。

### 32. Static Batching、Dynamic Batching、GPU Instancing 与 SRP Batcher 有什么区别？

**答案：**

“Batch”通常指一批提交或处理，并不是一种独立且唯一的技术。

| 技术 | 核心方式 | 适用条件与成本 |
| --- | --- | --- |
| 静态合批 | 组合适合一起绘制的静态几何数据 | 减少部分提交／状态开销，增加几何数据内存；需满足材质、Renderer 等兼容条件 |
| 动态合批 | CPU 变换和汇集足够小的网格 | 受顶点属性、材质、Pass、变换等条件限制；CPU 成本可能超过收益 |
| GPU Instancing | 同类网格和材质按实例数据绘制多份 | 适合同形植被、重复道具；Shader 需支持相应实例数据 |
| SRP Batcher | 减少兼容 Shader 变体之间的材质／状态准备工作 | 重点降低 CPU 提交成本，不等于把多个 Draw Call 变成一个 |

同场景可以使用多种技术，但同一 Renderer 的同一次绘制通常按兼容性与优先级选择，不能把收益机械相加。传统路径中静态合批、SRP Batcher、Instancing、动态合批有相应互斥或优先关系；Unity 6 的 GPU Resident Drawer／BRG 又有额外规则。

**纠错：**“缩放不同就一定不能动态合批”不成立。正负缩放造成的镜像差异是典型限制；还需查看网格属性数量等真实条件。HDRP 不支持传统动态合批。用 Frame Debugger 查看具体未合批原因。

参考：[官方资料 1](https://docs.unity3d.com/cn/2023.1/Manual/DrawCallBatching.html)；[官方资料 2](https://docs.unity3d.com/cn/2023.1/Manual/SRPBatcher.html)。

### 33. 什么是 Draw Call？如何优化而不牺牲整体性能？

**答案：**

Draw Call 是绘制命令，不是固定“一物体一次”。同一物体可能因子网格、Pass、阴影和多相机被多次绘制；一次实例化绘制也可以包含多个对象。

常见方案：共享兼容材质、合理使用合批或 Instancing、使用图集减少纹理切换、减少多余 Pass／相机、剔除不可见对象，并按距离使用 LOD。

合并 Mesh 会改变剔除粒度并可能增加无效绘制；同图集也不保证 UI 一次绘完，遮罩、排序、材质和 Canvas 都可能打断合批。Occlusion Culling 需要相应场景、烘焙／运行时配置，不是给任意物体加一个万能组件。

Draw Call 太多常增加 CPU 提交成本；GPU 是否繁忙还取决于三角形、像素、着色器、带宽等。以帧时间为目标，不只追求统计数字更小。

参考：[官方资料 1](https://docs.unity3d.com/cn/2023.1/Manual/optimizing-draw-calls.html)。

### 34. 什么是 Shader Variant？如何通过 keyword、strip 和 Warmup 优化？

**答案：**

Variant 是同一 Shader 在不同关键字、Pass、平台等条件下的程序变体。多个独立 keyword 组合会产生乘法增长，增加处理时间、包体、加载与运行时准备成本。

1. **Keyword 设计**：合并互斥选项，减少不必要组合，区分 `shader_feature` 与 `multi_compile` 的保留语义；local keyword 能减少全局关键字污染，但不会自动消灭组合数量。
2. **Strip**：按实际材质、平台、质量等级、管线功能裁掉不需要的组合；可使用管线配置及相应变体处理接口。动态切换、远程资源需要的组合要显式保留。
3. **Warmup**：在可控阶段准备确实会用的变体，例如使用 ShaderVariantCollection；用于降低首次使用卡顿，不负责减小包体。

过度剥离会导致缺失效果；一次性预热全部变体也可能造成长加载和内存峰值。不同图形 API 还涉及管线状态准备，不能把 Warmup 视为绝对消除全部首次绘制抖动的保证。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/shader-variants.html)；[官方资料 2](https://docs.unity3d.com/6000.0/Documentation/Manual/shader-loading.html)。

### 35. 修改 Material 为什么可能抖动？`material`、`sharedMaterial` 与 MPB 如何选择？

**答案：**

普通 `SetFloat`、`SetColor` 并不必然产生托管分配。要分别检查材质实例创建、托管临时对象、渲染状态变化和 GPU 数据更新。

- `Renderer.material` 在需要时为 Renderer 创建材质副本；不是每访问一次都必定再克隆。调用方应管理运行时材质的销毁。
- `sharedMaterial` 指向共享材质，修改会影响其他使用者；不能为了省实例而误改全局表现。
- 缓存 `Shader.PropertyToID`、材质引用和必要的临时容器，避免反复构造字符串、数组与新 Material。
- `MaterialPropertyBlock` 可传递逐 Renderer 属性并复用，但是否划算取决于渲染路径；它会破坏传统 SRP Batcher 兼容性，不能当作 URP／HDRP 的通用最优方案。

材质实例还涉及原生内存；托管 GC 不等于及时释放原生 Material。不同材质也不必然破坏 SRP Batcher，关键是兼容 Shader 变体和数据布局。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/Renderer-material.html)；[官方资料 2](https://docs.unity3d.com/cn/2023.1/Manual/SRPBatcher.html)。

### 36. 实时阴影如何实现？Shadow Map 的成本和优化点是什么？

**答案：**

常见 Shadow Mapping 先从光源视角渲染遮挡物深度，再在接收表面的着色过程中将片段深度与阴影图比较。

成本来自额外的遮挡物绘制、阴影图显存与带宽、过滤采样，以及多个灯光、级联或点光源方向带来的重复工作。

可减少实时投影灯光、缩短阴影距离、调整分辨率与级联、限制投射层和远处投影对象、使用简化投影网格，并对稳定内容采用烘焙或混合方案。

Bias 过小易出现自阴影痤疮，过大可能悬浮或漏光；必须结合尺度和光源调整。不能只降低阴影分辨率而忽略重复绘制和过滤开销。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/shadow-mapping.html)。

### 37. 如何判断 CPU Bound 或 GPU Bound？Profiler、Frame Debugger、RenderDoc 如何分工？

**答案：**

先在目标设备、可重复场景中采样，区分真实运算、同步等待、限帧和垂直同步，避免直接把 Editor 的结果当作发布环境结果。

| 工具 | 主要回答的问题 |
| --- | --- |
| Unity Profiler | 主线程、渲染线程、工作线程、GC、UI、物理分别花了多少时间；GPU 模块取决于平台支持 |
| Frame Debugger | 一帧有哪些绘制和 Pass，材质／纹理／目标如何切换，合批在哪里中断 |
| RenderDoc | 在支持的图形 API 上检查单帧资源、状态、纹理和绘制结果；不替代 CPU 分析 |
| Memory Profiler | 对象数量、引用关系、托管与原生内存变化；不是 GPU 时间的直接证据 |

**CPU 受限**：沿 Timeline 找脚本、物理、UI 重建、资源创建、渲染提交或等待热点，减少重复工作、分帧或并行纯计算。

**GPU 受限**：结合 GPU 时间降低 Render Scale、透明叠加、着色器复杂度、阴影和后处理做对照；区分像素、几何与带宽压力。降低分辨率明显改善帧时是线索，不是所有场景的充分证明。

一次只调整一个主要变量，记录 P50／P95 等帧时间和画质变化。

### 38. 常见光源和 Lightmap、Light Probe 有什么区别？

**答案：**

常见光源包括平行光、点光源、聚光灯与区域光，但可用形状和实时／烘焙支持取决于管线。不能脱离管线断言“只有四种”或“点光源永远最贵”。

Lightmap 将预计算的光照等结果存入纹理，适合相对稳定的表面；移动对象可通过适合的探针方案获得间接光照信息。点光源启用实时阴影时可能涉及多个方向的阴影渲染，成本还取决于分辨率、范围、物体数量和具体实现。

### 39. 多个相机同时活动时会怎样？Depth Only 是透明背景吗？

**答案：**

多个相机可能分别向同一目标、不同 RenderTexture 或不同显示器输出，最终效果由顺序、视口、目标与清除规则决定，不一定只是“画面混合”。

Built-in 的 Depth Only 主要清除深度而保留之前的颜色，常用于叠加相机；它不是自动让输出背景具有透明 alpha。URP 的 Base／Overlay Camera Stack 等有自己的兼容规则，应按当前管线组织。多相机也可能重复剔除、绘制、阴影和后处理。

### 40. OnBecameVisible／OnBecameInvisible 能当成玩家可见性判断吗？

**答案：**

这组回调反映 Renderer 对相关相机的可见状态变化，不等同于“玩家肉眼当前看见”。Editor 的 Scene 视图相机等也可能影响结果。

可用于部分表现层降频，但关键游戏逻辑不能只依赖它。需要特定相机的可见性、遮挡、屏幕范围或 AI 感知时，应使用对应的剔除与查询策略；回调也不是任意业务初始化与清理的唯一入口。

### 41. 把 GameObject 标成 Static 会发生什么？

**答案：**

Static 是一组供不同系统使用的静态标记，如批处理、光照或遮挡相关配置，具体选项随版本变化。它不是一个自动隐藏所有被遮挡物体的开关，也不应对所有“不常移动”的对象一键全选。

按系统实际需要设置，明确对象或层级是否会改变。静态合批、遮挡数据、烘焙光照各自有约束；业务后来移动对象时应重新评估这些假设。

### 42. Texture 与 Sprite 导入有什么区别？Sprite 必须是 2 的整数次幂吗？

**答案：**

Texture2D 保存纹理数据；Sprite 还描述纹理区域、Pivot、边框、网格等 2D 渲染信息。Sprite 导入模式可生成一个或多个 Sprite 子资源，用于 SpriteRenderer、UI Image 等。

Sprite 源图不普遍要求宽高为 2 的整数次幂。实际限制来自目标压缩格式、硬件、MipMap、图集与导入配置。图集能改善纹理共享，但不保证所有使用者合成一次绘制。

### 43. Terrain 使用 3、4、5 个地表纹理层，开销一样吗？

**答案：**

不能给出跨版本固定答案。每个 Pass 可处理的层数、额外 Pass、控制纹理、法线采样和远景合成策略取决于 Terrain Shader、管线与平台。

新增一层可能跨过单 Pass 容量而产生明显成本，也可能主要增加采样或资源占用。用 Frame Debugger 观察 Pass 与状态，用 GPU 时间和内存分析实际变化，不背“无论几层都只渲染一次”。

### 44. 如何保证 A 组先于 B 组绘制？绘制顺序等于最终遮挡关系吗？

**答案：**

先明确相同相机、相同目标还是跨相机。可按需求使用渲染队列、Sorting Layer／Order、相机叠加，或显式渲染 Pass。常见队列中较小值先处理，但同队列内部还可能按深度等排序。

绘制先后不等于最终覆盖关系：深度测试、深度写入、混合和模板测试都会影响结果。要保证特定效果，应同时定义这些状态；不要沿用“队列值越大越先画”的错误结论。

### 45. Surface Shader、顶点片段 Shader 与 Shader Graph 如何选择？

**答案：**

Built-in 的 Surface Shader 是较高层的着色描述方式，自动生成相关光照 Pass；不直接适用于 URP／HDRP。手写顶点／片段 Shader 控制力强，但需自行适配管线约定。

Shader Graph 是图形化创作方式，能力与目标管线和版本相关，最终仍生成对应着色程序。旧资料中的固定功能 Shader 分类属于历史上下文，不应作为现代管线能力的完整列表。

## 四、资源、依赖与内容更新

### 46. ScriptableObject 与普通 C# 类、Prefab 有什么区别？

**答案：**

ScriptableObject 是不依附 GameObject 的 Unity 对象，适合共享配置、编辑器数据和资源引用，可在编辑期保存为 `.asset`。它可以包含方法，不是只能存数据。

普通 C# 类适合业务运行态；符合规则的类可以通过 Unity 序列化嵌入其他对象，也可用自定义持久化，因此“普通类不能保存”不准确。Prefab 则主要保存 GameObject／组件层级模板。

配置可用 ScriptableObject，也可用表格导出的结构化数据，取决于规模和协作方式。应将共享配置与玩家、角色、Buff 的可变状态分开，避免运行时改动共享配置影响所有使用者；在 Player 中修改资产对象不等于自动保存到磁盘。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/class-ScriptableObject.html)。

### 47. AssetBundle 是什么？从资源发布到加载的流程是什么？

**答案：**

AssetBundle 是面向目标平台生成的资源容器，可包含序列化对象和相关资源数据，用于按需下载、分组加载和资源内容更新。扩展名可以自定义，不一定是 `.unity3d`，也不是通用代码热更新机制。

基本流程：规划资源分组与依赖 → 为目标平台产出 Bundle 和清单 → 发布带版本／哈希的文件 → 客户端校验并下载 → 先准备依赖 → 通过本地或网络加载接口读取 Bundle → 取得目标资源 → 按所有权释放。

本地可使用 `LoadFromFileAsync`，网络可使用 `UnityWebRequestAssetBundle`。资源发布流程可由 `BuildPipeline.BuildAssetBundles` 等编辑器 API 或上层系统组织。实际加载还需处理缓存、失败重试、峰值内存与卸载，不应只展示 `LoadAsset` 一步。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/AssetBundles-Dependencies.html)。

### 48. AssetBundle 的 Manifest、依赖加载与重复加载如何管理？

**答案：**

生成 Bundle 时会分析资源引用；对应 Manifest 记录 Bundle 哈希和依赖关系，运行时可用 `AssetBundleManifest.GetAllDependencies()` 查询所需依赖。

1. 以稳定的 Bundle 标识、内容版本建立加载表，缓存已完成结果并合并并发请求。
2. 在使用目标资源前确保依赖 Bundle 就绪；上层加载器统一维护依赖图和引用计数。
3. 加载成功再交付租约，失败时回滚本次已取得的引用。
4. 释放目标及依赖时逐级减引用，只有不再被其他使用者需要时才卸载。

AssetBundle 底层 API 不会替你完成整套依赖生命周期管理。同一资源打入多个 Bundle 可能产生重复资源；反复 `Unload(false)` 后重新加载也可能增加重复实例。循环依赖首先是分组与卸载粒度问题，不能仅凭存在环就断言必定泄漏。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/AssetBundles-Dependencies.html)。

### 49. 如何管理资源引用计数？Resources 与 AssetBundle 如何卸载？

**答案：**

引用计数表达“还有多少所有者在使用资源”，应与明确的获取／释放操作配对。它不是随便复制一个 C# 引用就自动变化的通用机制。

| 来源或操作 | 释放语义 |
| --- | --- |
| `Resources.Load` | 没有 Addressables 式自动租约计数；由业务管理引用，按资源类型使用 `UnloadAsset` 或在合适阶段使用 `UnloadUnusedAssets` |
| `AssetBundle.Unload(false)` | 释放 Bundle 容器相关数据，已加载资源仍可保留，需要另外管理其生命周期 |
| `AssetBundle.Unload(true)` | 同时卸载从该 Bundle 加载的对象，可能破坏仍在使用的资源引用 |
| 运行时 `new Material` 等 | 由创建方或转移后的所有者按对应 API 释放 |

`Resources.UnloadUnusedAssets` 是有成本的扫描操作，也不能替代应用自己的资源所有权设计。池中对象、静态缓存和其他资源引用都可能让资源继续存活。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/AssetBundle.Unload.html)。

### 50. `Destroy` 与 `DestroyImmediate` 有什么区别？

**答案：**

`Destroy` 将 Unity 对象安排为延迟销毁，常规运行时优先使用；销毁 GameObject 也涉及其组件和子层级。它不是把某个 C# 变量立刻赋成真正的 null。

`DestroyImmediate` 立即执行销毁，主要用于确实需要立即处理的编辑器流程。运行时不应频繁使用：在引擎回调或遍历过程中立即改变结构，可能导致状态与遍历问题。

资源应按其来源释放：Addressables 实例优先配对 `ReleaseInstance`，临时 RenderTexture 配对归还；不能把 `Destroy` 当作能平衡所有资源系统计数的万能接口。

### 51. Addressables 的 Handle、引用计数和依赖管理是什么？Release 有什么危险？

**答案：**

Addressables 加载返回 `AsyncOperationHandle`，用于观察状态、结果和异常，并代表需要管理的操作引用。资源及其依赖 Bundle 由系统维护引用计数，依赖关系通常是图，不一定是一棵树。

- 为每次获取明确所有者，使用结束配对 Release；复制句柄这个结构体本身不会增加引用计数。
- `LoadAssetAsync` 取得 Prefab 后自行 `Instantiate`，实例不会自动为该加载操作增加引用；实例仍依赖资源时不能提前释放唯一加载句柄。
- 使用 `InstantiateAsync` 创建的实例，按相应跟踪方式配对 `ReleaseInstance` 或释放实例操作句柄。
- 失败、取消等待、窗口关闭和场景切换也需要完成所有权收尾，不能只在成功路径 Release。

过早释放可能卸载仍在使用的资源；重复释放会破坏计数。计数归零也不等于进程内存立即下降，还受 Bundle 共驻、缓存等影响。频繁归零又重新加载会产生资源抖动。

参考：[官方资料 1](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/MemoryManagement.html)；[官方资料 2](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/AddressableAssetsAsyncOperationHandle.html)。

### 52. 资源热更新与 Lua／C# 逻辑热更新有什么区别？完整更新流程如何设计？

**答案：**

AssetBundle／Addressables 内容更新解决资源与数据替换；Lua 或特定 C# 运行方案解决行为逻辑更新，两者不是同一层。资源容器不会让任意新增 C# 程序自动可执行。

可落地的资源更新流程：

1. 客户端取得与应用版本、平台、资源格式兼容的版本清单。
2. 比较内容哈希与本地缓存，计算差量并检查空间。
3. 下载到临时文件，支持断点和重试，校验长度、哈希；需要防篡改时验证签名。
4. 所有必需资源准备完成后，原子切换清单或目录指针，避免半新半旧。
5. 通过版本化资源键加载；失败可回退到上一完整版本。
6. 只有没有活跃使用者和回退需求的旧内容，才进入缓存清理流程。

资源版本不应只靠一个整数比较；还需 schema、平台、依赖和应用兼容范围。更新内容也仍需满足发布平台要求。

### 53. 如何设计十万条配置数据的热更新系统？Lua 与 ScriptableObject 如何配合？

**答案：**

先区分数据规模、访问模式与逻辑热更新：十万条记录不应机械对应十万个独立 `.asset` 或 Lua 对象。

1. **生产链路**：表格／数据库作为源，校验主键、外键、类型和默认值，产出分表、分片的数据与 schema 版本。
2. **读取层**：通过稳定 ID 查询，建立常用索引，按模块懒加载；大数据优先考虑紧凑二进制、字符串表和批量解析。
3. **共享层**：配置快照只读共享，角色、Buff 等运行态另存；Lua 通过稳定接口访问，避免复制整表到多个堆。
4. **更新层**：下载并校验新快照，后台执行允许的解析，验证关联后在安全点切换版本；旧快照在无人使用后释放。
5. **兼容层**：处理 schema 迁移、缺失 ID、存档兼容、回滚与在途请求。

ScriptableObject 适合设计期编辑及资源引用，可由 Bundle 承载；Player 中修改它不等于持久化或更新类定义。评估首次加载、随机查询、分配量与双版本并存峰值。

### 54. Source File、Asset 与 UnityEngine.Object 是什么关系？

**答案：**

源文件是磁盘上的图片、模型、声音等输入；Asset 是项目中的资源概念，可来自 Assets、包或编辑器创建。导入后，一个源资源可能对应多个 Unity 对象，例如 FBX 的 Mesh、AnimationClip，或切片纹理的多个 Sprite。

`UnityEngine.Object` 是引擎对象体系的基类，加载的资源对象、场景对象与运行时创建对象都可属于它。不能说所有 Object 都是某个磁盘 Asset 的实例，也不能把磁盘文件与已加载对象一一对应。

多个 Renderer 引用同一 Sprite，通常共享该资源对象；实例化 GameObject 与加载资源是不同操作。

Player 中的资源文件布局属于平台与版本相关实现，不能保证场景数量与 `sharedassetsX.assets` 一一对应，也不应依赖这些内部文件名编写加载逻辑。图集是否生成取决于实际图集与交付配置，不由“直接引用还是 Resources”单独决定。

### 55. Asset Import Pipeline、.meta、Library 和 AssetDatabase 如何配合？

**答案：**

导入器把源文件、导入设置及依赖转换为目标平台适用的中间产物，通常缓存于 Library。`.meta` 保存 GUID 与导入相关设置，移动资源时应连同保留，不能靠重建 meta 保持身份。

导入结果还取决于 Unity／包／导入器版本、目标平台和依赖，不只是源文件与 meta 两项。AssetDatabase 是编辑器侧的资源操作 API，Player 中不能将它当作运行时加载器。

Library 通常可由项目输入重新生成，但重建有时间成本；备份项目的重点是源资源、meta、项目设置与包依赖，不是把缓存当成唯一源数据。

### 56. Unity 项目哪些目录应纳入版本管理？如何安全迁移资源？

**答案：**

通常保留 Assets 及其 `.meta`、ProjectSettings、Packages 中的依赖声明／锁定文件，以及项目实际使用的嵌入包等源码。Library、Temp、Logs 等生成内容通常不作为协作源数据。

迁移可通过包含依赖的 `.unitypackage`、可复用 UPM 包，或连同 meta 复制完整资源集合。核对目标项目的管线、包版本、脚本程序集与外部依赖。复制 Library 不是安全迁移的必要条件；Asset Server 属于旧工作流。

`.unitypackage` 是编辑期资源分发形式，不等同于 Player 运行时加载的 AssetBundle。使用 asmdef 后，脚本所在程序集也不能仅凭旧的 Plugins 目录规则判断。

### 57. 直接引用、Resources、StreamingAssets、AssetBundle 与 Addressables 如何选择？

**答案：**

| 方式 | 适用点与边界 |
| --- | --- |
| 场景／Prefab 直接引用 | 简单明确，依赖随内容交付；不必为了“动态”而替换所有引用 |
| Resources | 通过路径加载，相关资源纳入 Player 内容；纳入包体不等于启动时全部对象入内存 |
| StreamingAssets | 按约定交付原始文件，读取方式存在平台差异 |
| AssetBundle | 平台相关资源容器，加载与依赖生命周期需管理 |
| Addressables | 地址、Catalog、提供者、异步操作和引用管理的上层系统，常使用 AssetBundle |

Addressables 不是 AssetBundle 换名字后的新版本，也没有普遍取消底层 AssetBundle API。资源是否进入 Player、Bundle 或其他交付内容，取决于实际依赖和配置，不能只看是否在 Assets 下。

参考：[运行时资源加载](https://docs.unity3d.com/6000.0/Documentation/Manual/LoadingResourcesatRuntime.html)、[Addressables 概览](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/AddressableAssetsOverview.html)。

### 58. AssetBundle 如何分组？包越小越好吗？

**答案：**

综合资源的共同使用时间、共同卸载时间、更新频率、共享依赖和下载开销划分，而不是只按文件夹或资源类型机械分组。

经常一起使用的小资源可合包；真正被多处共享的重资源可独立，但过度拆分也会增加依赖、文件数和维护成本。大包会扩大下载与常驻粒度，小包也可能造成请求和元数据开销。

以实际报告检查重复依赖、包间引用、加载峰值和资源抖动。版本可用稳定内容哈希等管理，不能把扩展名或所谓 Variant 简单当作完整版本更新系统。

## 五、性能、GC 与对象池

### 59. Unity 是否按 Gen 0／1／2 回收？Mono 与 IL2CPP 的 GC 有何关系？

**答案：**

不能把桌面 .NET 的分代模型直接当作 Unity 结论。这里以 Unity 2022.3／6000.0 常见 Mono、IL2CPP 环境为范围：相关官方资料描述的是 Boehm 系列的**非分代、非压缩**回收器，而不是按 Gen 0／1／2 晋升的通用答案。

**增量 GC**把工作分散到多个时间片，降低单次长暂停的风险；它不等于分代，也不等于总回收工作消失。引用变化需要额外追踪，工作量大时仍可能出现明显回收成本。平台支持也有差异。

IL2CPP 改变代码的生成与执行方式，不会让托管分配自动变为手工释放，也不会消除 GC。回答时应先明确 Unity 版本、后端和平台，再讨论其实际 GC 配置。

参考：[官方资料 1](https://docs.unity3d.com/ru/2021.1/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html)；[官方资料 2](https://docs.unity3d.com/6000.0/Documentation/Manual/performance-incremental-garbage-collection.html)。

### 60. 如何定位 `Update()` 中的 GC Alloc？LINQ 有哪些风险？

**答案：**

GC Alloc 是托管分配量，不是本帧已经回收的字节数。先在目标设备上找到分配调用栈，再检查热点中的临时集合、数组、字符串、装箱、闭包和迭代器。

LINQ 的部分操作会创建迭代器、委托或物化结果；重复枚举还可能重复计算。不能断言所有 LINQ 都分配，也不能只看一行语法决定优化。

处理流程：记录基线 → 定位具体方法 → 复用缓存或预留容量 → 避免重复查询和临时对象 → 必要时用清晰循环替代 → 复测分配量与帧时间。Profiler 的 Deep Profile 等设置也会改变开销，需要区分定位工具本身的影响。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/performance-garbage-collector.html)。

### 61. 如何定位内存泄漏？怎样区分泄漏、缓存与内存峰值？

**答案：**

重复同一进入／退出场景或打开／关闭界面的流程，在稳定时点采集多次快照，比较对象数量、资源大小和引用链。结合 Unity Memory Profiler、运行时计数与平台工具分析托管、原生和图形内存。

常查项目：静态集合和事件、未结束异步流程、无上限对象池、未释放 Addressables 句柄、运行时 Material／Mesh／RenderTexture、原生容器与插件分配，以及动画系统中仍持有目标或回调的 Tween／Sequence。

纹理过大、Read/Write 保留额外副本通常首先是资源预算问题；Bundle 循环依赖可能让卸载粒度扩大；它们都不自动等于泄漏。DOTween 等第三方系统应检查完成／Kill／自动回收及回调持有关系，不能说每个 Sequence 都会泄漏。

托管堆容量或进程保留内存不下降，不能单独证明对象仍存活。要展示“哪些对象反复增加、由谁持有、何时本应释放”的证据。

### 62. 为什么托管包装对象被回收，不代表 Unity 原生资源已及时释放？

**答案：**

纹理、网格、RenderTexture 等资源可能同时涉及托管包装器和原生侧内存。GC 管理托管可达性，不能替代资源所有权及引擎资源的释放流程。

应根据资源创建方式使用对应的销毁、释放或归还机制，并检查资源是否仍被场景对象、缓存或对象池引用。分析 GC Roots、分代回收和 LOH 时，也不能直接将桌面 .NET 的全部实现细节套用到不同 Unity 版本、后端和平台。

### 63. 为什么大量零散 `Update()` 需要治理？有哪些替代方案？

**答案：**

问题在于无效轮询、分散调度、跨层调用和重复计算，不在于 `Update` 本身不可使用。简单且确实每帧需要的逻辑仍适合它。

可按事件驱动、脏标记、定时／分帧预算、集中调度、空间索引、状态机和 Job 并行拆分工作。集中调度器还应支持暂停、优先级、注册／移除和防止遍历中修改的规则。

`FixedUpdate` 用于物理步进，不是普通逻辑的性能替代品，低帧率时反而可能补跑多次；协程控频降低执行次数，但不会把计算挪到后台。事件驱动也要管理取消订阅，避免从轮询问题变成引用泄漏。

### 64. 如何优化每帧遍历大量实体的 C# 逻辑？

**答案：**

- 避免在循环中创建临时集合和闭包。
- 使用连续存储和顺序访问，改善缓存局部性。
- 将低频逻辑与高频逻辑分离。
- 对固定数据提前建立索引或缓存。
- 减少每个实体重复执行的组件查找。
- 使用 Profiler 确定 CPU 热点，再针对性优化。

当实体数量很大时，数据布局和访问模式可能比单纯减少几次方法调用更重要。

### 65. 对象池如何工作？预热、最大容量和状态重置有什么意义？

**答案：**

对象池维护可复用的空闲实例：取出 → 初始化本次使用状态 → 激活 → 使用 → 清理状态 → 归还。适合高频且创建成本明显的子弹、特效、列表项等。

- **预热数量**：提前承担一部分创建成本，防止首次高峰集中卡顿，但会增加加载时间与常驻内存。
- **最大空闲容量**：限制回收后保留多少实例；不是自动限制同时在用数量，需要的话另设总量预算。
- **重置流程**：清理事件、计时器、协程、Tween、异步回调、目标引用、粒子与刚体状态，防止跨次污染。

`SetActive(false)` 不销毁对象，也不会释放所有资源引用。池无上限、资源不归还、重复归还都可能出问题。低频大对象不一定值得入池，空闲池也需定时或按场景回收。

### 66. 手写对象池核心代码：如何实现创建、复用、归还和清理？

**答案：**

下面是主线程使用的最小 GameObject 池。约定 Prefab 根节点初始不激活；`Get()` 返回的对象也保持不激活，让调用者先设置本次参数再激活。不要在对象自己的 `OnDisable` 中再次调用 `Return()`。

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

public sealed class SimpleGameObjectPool : IDisposable
{
    private readonly GameObject prefab;
    private readonly Transform parent;
    private readonly int maxInactive;
    private readonly Stack<GameObject> idle = new Stack<GameObject>();
    private readonly HashSet<GameObject> leased = new HashSet<GameObject>();
    private readonly Action<GameObject> reset;
    private bool disposed;

    public SimpleGameObjectPool(GameObject prefab, Transform parent,
        int prewarm, int maxInactive, Action<GameObject> reset = null)
    {
        if (prefab == null || prefab.activeSelf)
            throw new ArgumentException("需要根节点未激活的 Prefab");
        if (prewarm < 0 || maxInactive < 1 || prewarm > maxInactive)
            throw new ArgumentOutOfRangeException(nameof(prewarm));
        this.prefab = prefab;
        this.parent = parent;
        this.maxInactive = maxInactive;
        this.reset = reset;
        for (int i = 0; i < prewarm; i++)
            idle.Push(UnityEngine.Object.Instantiate(prefab, parent));
    }

    public GameObject Get()
    {
        if (disposed) throw new ObjectDisposedException(nameof(SimpleGameObjectPool));
        GameObject item = null;
        while (idle.Count > 0 && item == null)
            item = idle.Pop();
        if (item == null)
            item = UnityEngine.Object.Instantiate(prefab, parent);
        leased.Add(item);
        return item; // 调用者先初始化，再 SetActive(true)
    }

    public void Return(GameObject item)
    {
        if (disposed) throw new ObjectDisposedException(nameof(SimpleGameObjectPool));
        if (ReferenceEquals(item, null) || !leased.Remove(item))
            throw new InvalidOperationException("对象不属于在用集合，或已重复归还");
        if (item == null) return; // 外部已销毁的 Unity 对象
        try
        {
            item.SetActive(false);
            reset?.Invoke(item);
            item.transform.SetParent(parent, false);
        }
        catch
        {
            if (item != null) UnityEngine.Object.Destroy(item);
            throw;
        }
        if (idle.Count < maxInactive)
            idle.Push(item);
        else
            UnityEngine.Object.Destroy(item);
    }

    public void Dispose()
    {
        if (disposed) return;
        disposed = true;
        foreach (GameObject item in idle)
            if (item != null) UnityEngine.Object.Destroy(item);
        foreach (GameObject item in leased)
            if (item != null) UnityEngine.Object.Destroy(item);
        idle.Clear();
        leased.Clear();
    }
}
```

`reset` 回调负责业务状态清理。`Dispose()` 会销毁包括在用对象在内的全部已登记实例，因此只在确定池的作用域结束后调用。该示例不处理异步资源加载和在用总量上限；若池依赖 Addressables 加载的 Prefab，仍要持有资源租约直到所有实例结束使用。

### 67. 为什么不应在热点中无节制调用 `GetComponent<T>()`？

**答案：**

组件查询需要执行查找工作。在高频路径中反复查询同一个稳定组件，会增加不必要的 CPU 开销。

```csharp
private Rigidbody _rigidbody;

void Awake()
{
    _rigidbody = GetComponent<Rigidbody>();
}
```

对于生命周期内不变的组件，可在初始化时缓存。是否值得优化应依据调用频率和 Profiler 结果，不必盲目缓存所有查询；组件发生变更时，也要维护缓存的有效性。

### 68. 移动端是否能照搬固定的内存、顶点、骨骼或灯光上限？

**答案：**

不能。设备档位、目标帧率、分辨率、管线和场景决定预算；“200 MB、8 万顶点、30 根骨骼”等旧项目数字不是行业通用阈值。

分别测量 CPU、GPU、托管／原生／图形内存、加载峰值和发热降频。根据瓶颈制定纹理压缩与分辨率、骨骼与蒙皮、粒子、光照、UI 和资源生命周期预算，并通过自动检查与真机回归维护。

隐藏对象或放入对象池会保留内存，不是释放内存；减面也不一定能解决像素填充受限。

## 六、异步、Job System 与数据并行

### 69. 为什么不能在子线程随意操作 `GameObject` 或 `Transform`？

**答案：**

Unity 大部分引擎对象 API 要求在主线程访问。在线程池中直接操作它们可能产生异常或未定义行为。

推荐流程：

1. 后台执行纯计算、支持后台执行的文件 I/O 或数据解析。
2. 将结果传回主线程。
3. 在主线程更新 `Transform`、`GameObject` 和 UI。

```csharp
Vector3 result = await Task.Run(() => CalculatePosition());

// 前提：此 await 已恢复到 Unity 主线程，且目标对象仍有效
transform.position = result;
```

`Task.Run()` 的委托通常在线程池执行，但 `await` 后是否回到主线程取决于上下文和等待器。若未捕获主线程上下文，或方法从后台线程开始执行，需要显式调度回主线程。

### 70. Unity 中如何设计一个线程安全的资源加载流程？

**答案：**

1. 后台线程执行允许在线程池运行的文件 I/O 和纯数据解析。
2. 将解析结果传递回主线程。
3. 在主线程或 Unity 明确支持的 API 环境中创建、更新引擎对象。
4. 使用任务取消、资源所有权管理和异常传播处理加载中断。
5. 避免多个任务重复加载同一资源，必要时使用共享任务缓存。

关键点是区分“数据加载”和“引擎对象创建”，不能把所有加载步骤都直接放进 `Task.Run()`。

### 71. Unity 中如何避免异步加载完成后访问已销毁对象？

**答案：**

- 在对象销毁或禁用时取消关联任务。
- 使用取消令牌或生命周期标记检查对象是否仍有效。
- 在异步结果应用到场景前，重新验证目标对象。
- 对共享资源加载任务，区分单个调用者取消和共享任务整体取消。

仅仅取消等待者，并不一定会停止底层加载工作。

对象池复用时，还应校验本次请求的版本号或租约 ID：对象可能仍存在，但已被另一轮业务使用。取消等待与终止底层加载是两件事，完成后仍需要处理过期结果的释放。

### 72. Unity 的 `async/await` 有哪些坑？continuation 为什么看起来不执行？

**答案：**

先确认等待对象是否已完成，再看恢复过程；“没有继续执行”不一定就是死锁。

1. 用 `.Result`／`.Wait()` 阻塞主线程，而后续又需要主线程同步上下文，可能形成死锁。
2. 包装回调时，`TaskCompletionSource` 没有在成功、失败、取消等路径正确结束，任务会一直等待。
3. 场景卸载、退出 Play Mode、暂停或自定义 PlayerLoop 变化，可能使某些依赖循环的调度失效。
4. `async void` 难以被调用方等待和统一观察异常；优先返回 `Task` 或合适的 awaitable。
5. 从后台进入、使用 `ConfigureAwait(false)` 或切换等待器后，不能无条件假设返回主线程。

Unity 的 Task 与 `Awaitable` 有不同恢复语义。Unity 6 的 Awaitable 使用池化机制，不应多次 await 同一实例；还需考虑完成回调可能同步恢复带来的重入。

定位时记录操作 ID、起止状态、线程、取消原因和异常，给外部操作设置合理超时，并确保超时后仍处理最终资源归属。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/async-await-support.html)；[官方资料 2](https://docs.unity3d.com/cn/6000.0/Manual/async-awaitable-continuations.html)。

### 73. Job System 与 Burst 分别做什么？NativeContainer 有哪些约束？

**答案：**

**Job System** 将符合规则的工作调度到工作线程，并用 JobHandle 表达依赖；**Burst** 对其支持的 C# 子集和数据模式生成优化的原生代码，利用 LLVM 优化、向量化等能力。二者可以配合，但不是同一系统。

性能通常来自连续数据、较少托管间接访问、SIMD 与并行；不是加一个属性就一定提速。任务过小、频繁 Schedule 后立刻 Complete、共享写竞争和数据搬运都可能抵消收益。

NativeContainer 持有原生内存，复制容器结构体通常仍指向同一份存储。应声明正确的读写权限、遵守依赖、选择合适分配器，并在相关 Job 完成后释放；`TempJob` 的使用期限等规则需遵守官方约束。不要禁用安全检查来掩盖数据竞争。

普通 Job 不能随意调用 GameObject API；先处理数据，再在允许的线程应用结果。

参考：[官方资料 1](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/index.html)；[官方资料 2](https://docs.unity3d.com/6000.0/Documentation/Manual/job-system-native-container.html)。

### 74. 高性能 AOI 系统选择 Job + NativeContainer，还是 DOTS ECS？

**答案：**

这不是完全互斥的选项：DOTS ECS 也常配合 Jobs、Burst 和原生数据。选型取决于项目已有架构、实体规模、更新频率和迁移成本。

推荐先回答算法：采用均匀网格／空间哈希或适合分布的层级索引，把全量两两比较改为邻近候选查询；按进入／离开阈值设置滞回，避免边界抖动。

- 现有 GameObject 项目：用连续位置数组和稳定 ID，Job 并行生成兴趣变化，主线程消费进入／离开事件，改造成本较小。
- 已经数据导向且实体很多：ECS 便于按组件批量处理，但要关注结构变化、Chunk 布局、共享组件划分与同步点。

输出应采用线程安全的分区写入或并行容器，处理 ID 复用和销毁；测量热点区域聚集、移动突发、候选数量、结果合并和网络订阅成本，而非只看理论复杂度。

## 七、架构与系统设计

### 75. 如何设计支持异步、缓存、引用计数与复用池的 UI 加载系统？

**答案：**

可划分为资源层、实例池、窗口路由与表现层，避免一个“UIManager”承担所有职责。

1. 用稳定 UIKey 映射 Prefab 地址、层级、缓存策略和窗口类型。
2. 同一资源合并并发加载；资源层维护加载状态及明确的引用租约。
3. 打开窗口时先查实例池，否则异步取得 Prefab 并在主线程实例化；参数和状态设置后再显示。
4. 每次打开生成请求 ID，关闭、路由变化或场景退出后，过期完成结果不得重新打开窗口。
5. 关闭时解绑事件、取消本窗口等待、清理动画与临时引用，再归池或销毁。
6. 缓存设置容量、超时和内存压力淘汰；池中实例仍依赖 Prefab 资源，不能因窗口“不可见”就过早归零资源计数。

共享加载只取消某个等待者，不应随便取消其他窗口的加载。失败、重试、重复打开和关闭中完成都应有明确状态转换。

### 76. 如何设计跨场景 ECS-like 组件系统？

**答案：**

这是架构方案，不是唯一引擎标准。可将稳定业务实体身份与 GameObject 表现分离：

- **Entity**：使用索引加 generation 的句柄，防止实体销毁后旧 ID 误命中新对象。
- **Component Store**：按类型保存连续数组、稀疏集合或适合查询的数据，不把所有状态塞进对象字典。
- **System**：声明读写数据和执行顺序，集中处理符合组件组合的实体。
- **World／Scope**：区分全局实体与场景实体；卸载场景时清理所属 scope 的实体、订阅和资源租约。
- **View Bridge**：按需把实体数据同步到 GameObject，表现对象可池化，逻辑实体不必跟随场景对象销毁。

结构变更通过命令缓冲在安全点应用，异步结果校验 generation。序列化保存业务 ID，不使用 InstanceID 作为跨会话主键。不要仅因名称类似 ECS 就复制整套 DOTS 复杂度。

### 77. Unity 中如何落地 MVC／MVVM？怎样避免复杂度失控？

**答案：**

| 层次 | 主要职责 |
| --- | --- |
| Model | 业务数据、规则和用例，不直接依赖具体 UI 控件 |
| View | 展示与输入采集，适配 GameObject、UGUI 或其他 UI 实现 |
| Controller／Presenter | 协调输入、业务调用和刷新，适合显式流程 |
| ViewModel | 提供供 View 绑定的状态和命令，适合较复杂的数据驱动页面 |

MVC／MVP 的风险是控制层变成大类、手动刷新遗漏；MVVM 的风险是通知链、双向绑定循环、订阅泄漏及高频通知引起布局重建。

优先单向数据流、按页面划分 scope、批量合并通知、明确订阅释放；少量静态页面可以直接 Presenter，不必给每个按钮都建立多层抽象。将资源加载、路由和业务规则隔离，避免 ViewModel 直接掌控所有 Unity 对象。

### 78. 如何设计支持叠层、扩展行为、事件与 UI 的 Buff 系统？

**答案：**

把**共享定义**与**运行实例**分开。

- BuffDefinition：ID、图标、持续时间、最大层数、互斥组、效果列表、触发条件，只读共享。
- BuffInstance：来源、目标、层数、剩余时间、累计 Tick 和本次订阅句柄，属于具体角色。
- StackingPolicy：分别定义增加层数、刷新时间、延长时间、独立计时和替换强度，不能用一个分支混合所有规则。
- Effect／Trigger：效果通过组合接口扩展；复杂触发可接入行为树或条件图，简单加成无需强行使用行为树。
- Attribute Pipeline：明确基础值、加法、乘法、上限等计算顺序；按句柄移除贡献，避免反复加减造成漂移。

战斗逻辑发出 BuffAdded／StackChanged／Removed 等事件，UI 只订阅展示，不成为计时权威。遍历中新增／移除可进入命令队列，处理触发递归、死亡清理、来源销毁和读档恢复。联机场景还需明确服务器权威与时间同步。

## 八、脚本后端与原生交互

### 79. Mono 与 IL2CPP 如何执行 C#？Unity 跨平台究竟意味着什么？

**答案：**

通用流程可概括为：

```text
C# → 托管程序集（IL + 元数据）
  ├─ Mono：由目标平台的运行时执行，常见桌面场景采用 JIT
  └─ IL2CPP：IL → C++ → 目标平台原生机器码，并配合运行时支持库
```

IL2CPP 的优势在于 AOT 部署能力、目标平台工具链优化和平台覆盖；代价包括较长的代码处理时间、更复杂的原生调试，以及动态代码生成、反射和裁剪方面的约束。它仍需要 GC、类型、异常与线程等运行时服务，并非“转成 C++ 就没有托管语义”。

跨平台意味着业务代码可以在不同平台复用并生成相应产物，不是同一二进制天然运行在所有设备。原生插件、资源格式、图形 API、输入与文件系统仍有平台差异。

不能保证 IL2CPP 恒定快 1.5～2 倍，也不能仅凭后端名称判断包体、启动时间或所有 C# 特性支持情况。语言版本、API 配置、平台政策和后端能力是不同维度，旧版本平台支持列表应按实际项目重新核对。

参考：[官方资料 1](https://docs.unity3d.com/6000.0/Documentation/Manual/il2cpp-introduction.html)。

现代 .NET 本身也支持跨平台，不能沿用“.NET 只能运行于 Windows”的旧结论。通用渲染流水线与坐标变换见 [图形学与3D数学题库](图形学与3D数学题库.md)。

### 80. JIT、AOT 与解释执行有什么区别？为何影响泛型、反射和热更新？

**答案：**

- **JIT**：运行时将需要的中间代码转成机器码。
- **AOT**：运行前生成目标机器码；不意味着运行时还“必须有一部分 JIT”。
- **解释执行**：运行时解释字节码等表示，不必为每段逻辑生成原生代码。
- **混合方案**：根据平台与框架，把提前生成的代码与解释器等组合；不是所有 Lua、Python 或热更新方案都只能归为单一实现。

IL2CPP 环境仍支持许多反射用法，但运行时动态代码生成受到限制。间接引用的类型／成员可能被裁剪，某些泛型使用还需要确保目标实现可用；`link.xml`、`[Preserve]` 和明确的泛型调用路径分别解决相应问题，不能一项配置包治全部。

平台对运行时生成可执行代码的限制，也不能说成“机器码不能映射到内存”——正常应用本身就在执行机器码。

### 81. Unity 的 C# 包装对象如何与 C++ 原生对象关联？GCHandle 是必需的固定步骤吗？

**答案：**

先区分托管包装器和原生引擎对象，它们不是同一块内存，也不保证同时销毁。

以 Unity 6000.0 公开绑定源码为例，`UnityEngine.Object` 包含 `m_CachedPtr`、`m_InstanceID` 等信息，绑定／封送层据此访问原生对象；这些是版本相关实现线索，不应扩写成完整的私有对象表规范。

| 概念 | 应如何回答 |
| --- | --- |
| Native Object | 引擎侧对象，生命周期由对应引擎与资源机制管理 |
| InstanceID | 当前会话对象身份，供部分定位、比较等逻辑使用，不是持久业务主键 |
| GCHandle | 托管与原生互操作的句柄机制；Normal 可保活，Pinned 还涉及固定地址，二者不能混称 |
| Binding／Marshalling | 负责调用和数据转换，不能把所有引擎 API 都统一描述为普通 P/Invoke |

**重要纠错：**不能断言所有 Unity 对象都一一创建 Pinned GCHandle，或存在一个普遍实现 IDisposable 的 NativeObject 基类。普通 UnityEngine.Object 没有统一 Dispose 契约，应按 Destroy、资源系统 Release 或特定类型 Dispose 等具体机制处理。具体原生回调如何持有托管对象，需看对应后端和类型实现。

参考：[官方资料 1](https://raw.githubusercontent.com/Unity-Technologies/UnityCsReference/6000.0/Runtime/Export/Scripting/UnityEngineObject.bindings.cs)。

### 82. 如何设计支持脚本热重载的系统？与编辑器 Domain Reload 有何区别？

**答案：**

运行中热重载需要有明确的替换边界：停止旧脚本更新与在途回调 → 保存可迁移的状态 → 加载或切换新逻辑 → 重建脚本实例 → 恢复数据与引擎对象绑定 → 清理旧委托、订阅和资源。

状态迁移应使用稳定 ID 和带版本的数据模型，处理新增／删除字段、失败回滚和正在执行的异步操作。不能把旧实例直接当成新类型实例复用。

普通 .NET 的可卸载 AssemblyLoadContext、Mono 的域机制、Unity 编辑器 Domain Reload 和 IL2CPP Player 是不同环境。编辑器脚本重新加载不自动提供发布端热更新能力；具体方案可采用脚本解释器、桥接层或适配目标平台的混合运行方案，并检查泛型、桥接与元数据限制。

如果无法卸载旧逻辑，必须明确隔离与内存上限，不能只替换一个程序集路径就宣称完成热重载。

## 九、动画、平台与旧项目维护

### 83. 骨骼动画、刚性部件动画与顶点动画有什么区别？

**答案：**

刚性部件动画直接变换独立部件；骨骼动画用层级骨骼变换和蒙皮权重影响网格顶点；顶点／形变动画可直接记录或驱动顶点变化，BlendShape 是常见形变表达之一。

线性蒙皮可概括为：每个顶点由若干骨骼变换后的结果按权重加权，结合绑定姿态转换。骨骼数量、每顶点影响数、动画采样、混合与蒙皮顶点量都会产生成本。

Root Motion、IK、重定向和物理驱动需明确职责；“关节动画”这个名称不能代替对数据与实际变形方式的说明。

### 84. Animator 有大量状态或大量角色时如何优化？

**答案：**

区分图的复杂度与同时活跃的采样／混合工作。可用子状态机、共享 Controller、参数 Hash 和明确的状态规则改善维护；需要程序化组织时评估 Playables，而不是一味增加状态与跨层过渡。

性能上查看层数、活跃混合、IK、蒙皮、骨骼数量和频繁初始化。按可见性与距离降频或使用适合的 Culling Mode，但检查 Root Motion、事件和关键逻辑是否依赖动画更新。

移除无意义 Animator 有价值，不等于旧 Animation 组件在所有场景都更合适。

### 85. Unity 与 Android／iOS 原生代码如何交互？

**答案：**

Android 常通过 AndroidJavaObject／AndroidJavaClass 与 Java／Kotlin 桥接，底层涉及 JNI；iOS 原生插件常导出 C ABI，再通过适当的互操作声明调用，Objective-C／Swift 可在原生层封装。

明确调用线程、参数编码、对象引用与回调生命周期。原生回调未必在 Unity 主线程，应用引擎对象前应正确调度；释放托管委托前要保证原生端不再回调。UnitySendMessage 可用于部分简单消息，但不是所有交互的唯一入口。

### 86. 【历史题】Unity 支持哪些脚本语言？怎样回答 Unity 与 Cocos 的比较？

**答案：**

现代 Unity 的主要用户脚本语言是 C#；UnityScript 和 Boo 是历史支持，不能按旧题库列为现代默认选项。Lua 通常通过项目选用的第三方运行或桥接方案接入。

引擎比较先明确产品与版本：Cocos2d-x 与 Cocos Creator 不是同一个开发栈。按 2D／3D 需求、平台、渲染与工具链、团队经验、原生扩展和维护成本比较。授权、费用和平台清单会变化，不沿用“永远免费”或旧 Flash／Wii 支持列表。

具体插件、语言版本与发布政策应查目标项目文档。

## 易错结论速查

| 容易背错的说法 | 更准确的理解 |
| --- | --- |
| 每帧都先执行一次 FixedUpdate | 一帧可能没有或有多次固定更新 |
| 销毁脚本组件不会停止协程 | 销毁承载组件会停止；仅禁用组件不停止 |
| 碰撞双方都必须有 Rigidbody | 静态 Collider 可以与动态刚体碰撞 |
| SRP Batcher 把所有物体合成一次 Draw Call | 重点是减少 CPU 状态准备和提交成本 |
| 改材质属性一定产生 GC | 分别检查实例创建、托管分配、原生资源和渲染更新 |
| Unity GC 一定按 Gen 0／1／2 晋升 | 不可直接套用桌面 .NET 分代模型 |
| Addressables Release 只需成功时处理 | 取消、失败与生命周期退出也需要配对管理 |
| GUID 永久不变、InstanceID 可写入存档 | GUID 依赖 meta 保留；InstanceID 不跨会话稳定 |
| 所有 Unity 对象都由 Pinned GCHandle 固定 | 不能把局部互操作方式概括为所有引擎对象的实现 |
| IL2CPP 永远更快且不再需要 GC | 性能取决于场景，托管内存仍需运行时管理 |
| Resources 全部资源启动即入内存 | 包含于交付内容与对象实际加载时机是两件事 |
| Addressables 已取消 AssetBundle | Addressables 通常在其上提供地址、依赖与引用管理 |
| Sprite 源图必须是 2 的幂 | 约束取决于导入、压缩与目标硬件 |
