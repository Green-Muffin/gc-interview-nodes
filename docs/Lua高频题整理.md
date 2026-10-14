# Lua 高频面试题与答案

> 语言、对象模型、GC、跨语言生命周期与常见性能题；明确 Lua 5.1／5.4 及 LuaJIT 差异。 共 10 题。

本专题以概念、机制、边界和实战追问组织；示例可能省略外围业务代码。

[返回题库总目录](面试题库总目录.md)

## 目录

| 主题 | 题号 |
| --- | --- |
| [一、语言与对象模型](#一、语言与对象模型) | 1–4 |
| [二、内存与跨语言生命周期](#二、内存与跨语言生命周期) | 5–8 |
| [三、排序与字符串](#三、排序与字符串) | 9–10 |

## 一、语言与对象模型

### 1. Lua 有哪些基本类型？哪些值是假？

**答案：**

Lua 5.1／5.4 都有八种基本类型：nil、boolean、number、string、function、userdata、thread、table。只有 nil 和 false 在条件判断中为假，0 与空字符串都为真。

Lua 5.3／5.4 的 number 区分整数与浮点子类型；Lua 5.1 和 LuaJIT 的具体数值实现要另看配置。`thread` 通常表示 Lua 协程状态，不等于操作系统线程。

参考：[Lua 5.4 类型](https://www.lua.org/manual/5.4/manual.html#2.1)。

### 2. table 的数组部分、nil、长度与遍历有什么坑？

**答案：**

table 可同时作为序列和映射，但有空洞的数值键集合不应当作连续数组使用。给一个键赋 nil 会移除对应条目；不要用 `#t` 统计任意字典的元素数。

`ipairs` 适合从 1 开始的连续序列，遇到首个缺失位置停止；`pairs` 用于键值遍历，不保证普通 table 的业务排序。需要稳定顺序时显式维护键数组并排序。元方法与版本也可能影响遍历接口。

### 3. Lua 如何用 table 和元表实现面向对象？`.` 与 `:` 有什么区别？

**答案：**

Lua 不要求内建 class 关键字。常用 table 保存实例状态，用元表的 `__index` 查找共享方法；继承可组织为查找链，多态由方法查找与替换实现。

```lua
local Actor = {}
Actor.__index = Actor

function Actor.new(hp)
    return setmetatable({ hp = hp }, Actor)
end

function Actor:damage(amount)
    self.hp = math.max(0, self.hp - amount)
end

local actor = Actor.new(100)
actor:damage(10) -- 等价于 actor.damage(actor, 10)
```

把可变实例数据放在实例里，避免原型上的共享 table 意外让所有实例共用状态。元表实现复用，不自动提供所有语言级访问控制。

参考：[Programming in Lua：对象](https://www.lua.org/pil/16.html)。

### 4. 闭包与 upvalue 是什么？为何定时器容易保留对象？

**答案：**

闭包可访问词法作用域中的外部局部变量，这些被引用的变量通常称为 upvalue。回调被定时器、事件表或全局表长期保存时，也可能让其捕获的对象继续可达。

不要把捕获理解为创建时的值快照，也不应只在函数返回时认为捕获对象已释放。停止业务流程时解除回调，区分共享配置、实例状态与长期缓存。

## 二、内存与跨语言生命周期

### 5. Lua 的 GC 是什么机制？不同版本能用同一套答案吗？

**答案：**

不能忽略版本。Lua 5.1 官方手册描述增量标记清扫；Lua 5.4 提供增量和分代模式。LuaJIT 也不能仅凭 Lua 语法兼容就视为使用同一实现。

GC 追踪可达性，孤立循环引用一般可以回收；仍被根引用或注册表持有的对象不会因为“业务不用了”自动消失。调 GC 参数需要结合分配率、存活量与停顿测量，不应频繁强制完整回收掩盖生命周期问题。

参考：[Lua 5.1 GC](https://www.lua.org/manual/5.1/manual.html#2.10)、[Lua 5.4 GC](https://www.lua.org/manual/5.4/manual.html#2.5)。

### 6. Lua 内存持续上涨，如何定位泄漏？

**答案：**

重复同一业务进入／退出流程，在稳定时点记录内存与对象数量。排查全局表、module 缓存、事件／计时器、未结束协程、闭包 upvalue、registry 引用和 userdata 的持有关系。

可结合对象标记、引用图、调试库与弱表观察定位保活路径，但遍历诊断本身也可能临时持有对象。仅看到 `collectgarbage("count")` 不下降不能直接证明原生资源泄漏；跨 C#／C 的桥接还需分别观察各侧堆。

修复应明确哪个所有者本应解除引用，而不是到处把变量设为 nil。

### 7. 弱表有什么用途？能自动解决所有缓存泄漏吗？

**答案：**

元表中的 `__mode` 可设置弱键 `k`、弱值 `v` 或同时弱 `kv`。弱引用不会像普通强引用那样单独阻止相应对象回收，适合某些辅助索引与缓存。

弱表内容可能消失，使用时必须处理缺失；它也不能替代外部资源的确定性释放。弱键与值之间的关联语义、字符串等特殊值行为要按 Lua 版本理解，不要假设所有键和值都以相同方式被回收。

参考：[Programming in Lua：弱表](https://www.lua.org/pil/17.html)。

### 8. Lua 与 C# 互相持有对象时如何管理生命周期？

**答案：**

桥接可能用 userdata、注册表引用、托管委托或句柄关联两侧对象。若 Lua 保存 C# 回调，而 C# 又长期持有 Lua 函数包装器，单侧 GC 未必能独立识别整体业务生命周期。

建立显式订阅／解除、Acquire／Release 或 Dispose 契约，先停止回调再释放桥接引用；业务退出时清理定时器与事件。不要把某个框架的包装器实现视为所有 Lua 插件的保证，也不要假定 Lua GC 会自动释放全部 Unity 原生资源。

## 三、排序与字符串

### 9. `table.sort` 用什么算法？稳定吗？比较函数有什么要求？

**答案：**

标准 Lua 5.4 的 `ltablib.c` 使用快速排序体系的实现；具体实现属于版本细节，不能推广到所有 LuaJIT 或定制运行时。API 不保证稳定排序。

比较函数应表达“a 严格排在 b 前面”，例如 `a.score < b.score`，不能写成 `<=`。相等元素之间应同时返回 false，规则还应保持传递一致。

需要可重现的同分顺序时，可用原始索引等作为第二关键字。排序作用于序列区间，返回值不是一个新 table。

参考：[Lua 5.4 table.sort](https://www.lua.org/manual/5.4/manual.html#pdf-table.sort)、[官方排序实现](https://www.lua.org/source/5.4/ltablib.c.html)。

### 10. Lua 如何拼接字符串？大量拼接如何处理？

**答案：**

`..` 用于连接字符串。字符串不可变，循环中不断把长结果与新片段连接，可能重复复制已有内容。

```lua
local parts = {}
for i = 1, 1000 do
    parts[#parts + 1] = tostring(i)
end
local text = table.concat(parts, ",")
```

大量片段可先保存到连续数组再 `table.concat`，并注意峰值内存；少量表达式拼接不必全部改写。nil 不是可直接放进 concat 区间的字符串元素。

参考：[Lua 字符串连接](https://www.lua.org/pil/3.4.html)、[table.concat](https://www.lua.org/manual/5.4/manual.html#pdf-table.concat)。
