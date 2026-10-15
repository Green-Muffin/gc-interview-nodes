# Android 与 JVM 客户端面试题

> 共 7 题。按主题分类，答案直接放在题目后。

[返回题库总目录](面试题库总目录.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、代码生成、反射与类加载](#%E4%B8%80%E3%80%81%E4%BB%A3%E7%A0%81%E7%94%9F%E6%88%90%E3%80%81%E5%8F%8D%E5%B0%84%E4%B8%8E%E7%B1%BB%E5%8A%A0%E8%BD%BD) | 1–3 |
| [二、协程与内存诊断](#%E4%BA%8C%E3%80%81%E5%8D%8F%E7%A8%8B%E4%B8%8E%E5%86%85%E5%AD%98%E8%AF%8A%E6%96%AD) | 4–5 |
| [三、UI](#%E4%B8%89%E3%80%81UI) | 6–7 |

## 一、代码生成、反射与类加载

### 1. KSP 在什么时候工作？与运行时反射有什么区别？

**答案：**

KSP 在构建期间读取 Kotlin 源码的声明、类型和注解等符号，并生成新的源码或资源；生成结果随后与项目代码一起处理。它不能检查任意函数体表达式，也不能直接修改原源码。

运行时反射通过 Class 等元数据查询类型、构造器、字段、方法和具有运行时保留策略的注解，灵活但有查找、动态调用和裁剪兼容成本。已知结构优先代码生成，高度动态的插件／调试场景再考虑反射。

### 2. 运行时反射读取注解需要满足什么条件？

**答案：**

注解必须允许用于目标元素，并采用 RUNTIME 保留策略，才可通过运行时反射读取。SOURCE 只存在于源码处理阶段，CLASS 会写入 class 文件但不要求 JVM 运行时暴露。

还要考虑注解是否继承、可否重复、读取的是类／方法／字段还是参数。混淆和代码裁剪可能移除仅靠反射访问的类型或成员，需要对应保留规则。

### 3. JVM／Android 类加载过程是什么？自定义 String 会覆盖系统 String 吗？

**答案：**

类加载器按类名查找字节码／DEX，经过加载、验证、准备、解析和初始化等阶段形成 Class；通常先委托父加载器，使核心类由 Bootstrap 等上层加载器定义，避免应用替换平台核心类型。

应用可以在自己的包中定义同名 `String`，但不会替换 `java.lang.String`；完整限定名不同。试图定义受保护的 `java.*` 核心包也不会正常覆盖平台实现。类的身份还包含定义它的 ClassLoader，同名类由不同加载器定义时可视为不同类型。

## 二、协程与内存诊断

### 4. Kotlin 协程异常如何传播和处理？

**答案：**

结构化并发中，普通 child 的非 CancellationException 失败通常取消父 Job，并连带取消兄弟。`async` 将异常保存在 Deferred 中，在 `await()` 时重新抛出；`launch` 的未处理根异常交给 CoroutineExceptionHandler／平台最终处理。

`SupervisorJob`／`supervisorScope` 让子任务独立失败，适合部分任务失败不影响其他任务。业务可恢复异常在协程内部 try/catch；不要吞掉 CancellationException，作用域应绑定 Activity、ViewModel 等生命周期并及时取消。

### 5. LeakCanary 如何检测内存泄漏？

**答案：**

它在 Activity、Fragment View 等生命周期结束时用弱引用观察本应被回收的对象。等待并触发检查后若对象仍被保留，就生成 heap dump。

Shark 分析对象图，从 GC Root 找到阻止该对象回收的强引用路径，生成 leak trace。对象暂时 retained 不一定就是业务泄漏，需结合生命周期和引用链判断；heap dump 本身会暂停应用，只用于诊断。

## 三、UI

### 6. Compose 与 XML View 系统的渲染模型有什么区别？

**答案：**

XML View 系统通常膨胀出有状态 View 树，代码通过 setter 修改节点；Compose 用 Composable 根据状态声明 UI，读取的 State 改变时安排重组，并尽量跳过输入稳定且未变化的部分。

Composable 可能多次、无固定顺序执行，应保持快速、幂等并把副作用放入 Effect。Compose 减少手动同步 UI 状态的代码，但重组范围、状态稳定性和频繁分配仍会影响性能；两套系统可通过互操作逐步迁移。

### 7. Android 自定义 View 的 measure、layout、draw 流程是什么？

**答案：**

`onMeasure` 根据父级 MeasureSpec 和内容计算测量尺寸，并调用 setMeasuredDimension；父 ViewGroup 在 `onLayout` 为子 View 分配位置；`onDraw` 使用 Canvas 绘制自身内容，ViewGroup 还负责子节点绘制。

`requestLayout()` 请求重新测量／布局，`invalidate()` 请求重绘。不要在 onDraw 中频繁创建 Paint、Path 等对象；触摸处理和可访问性也应纳入自定义控件设计。

## 参考资料

- [Android：KSP](https://developer.android.com/build/migrate-to-ksp)
- [Kotlin：协程异常](https://kotlinlang.org/docs/reference/coroutines/exception-handling.html)
- [LeakCanary 原理](https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/)
- [Android：Compose 思维模型](https://developer.android.com/develop/ui/compose/mental-model)
- [Android：自定义 View](https://developer.android.com/develop/ui/views/layout/custom-views/custom-components)
