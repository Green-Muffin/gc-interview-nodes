# Lua 面试题与答案

> 共 15 题。按主题分类，答案直接放在题目后。

[返回题库总目录](面试题库总目录.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、语言与对象模型](#%E4%B8%80%E3%80%81%E8%AF%AD%E8%A8%80%E4%B8%8E%E5%AF%B9%E8%B1%A1%E6%A8%A1%E5%9E%8B) | 1–6 |
| [二、GC 与生命周期](#%E4%BA%8C%E3%80%81GC%20%E4%B8%8E%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F) | 7–10 |
| [三、常见函数与协程](#%E4%B8%89%E3%80%81%E5%B8%B8%E8%A7%81%E5%87%BD%E6%95%B0%E4%B8%8E%E5%8D%8F%E7%A8%8B) | 11–15 |

## 一、语言与对象模型

### 1. Lua 有哪些基本数据类型？哪些值为假？

**答案：**

八种基本类型是 nil、boolean、number、string、function、table、userdata 和 thread。条件判断中只有 nil、false 为假，0 和空字符串都为真。

Lua 5.3／5.4 的 number 有整数和浮点子类型；thread 表示 Lua 协程，不是操作系统线程。

### 2. Lua table 的底层结构和特点是什么？

**答案：**

标准实现通常结合数组区和哈希区，适合的正整数键进入数组区，其他键通过哈希查找。table 可表示序列、字典、对象和模块。

赋 nil 会删除键；有空洞的表不适合当连续序列，`#t` 不能用于统计任意字典条目。重哈希可按分布调整两区大小。

### 3. pairs 和 ipairs 有什么区别？

**答案：**

pairs 遍历表中的键值，普通 table 不保证业务排序；ipairs 从索引 1 开始依次读取，到第一个缺失位置停止。

连续数组可用 ipairs，字典用 pairs；需要固定顺序时维护键数组并排序。

### 4. 元表和元方法是什么？__index、__newindex 如何使用？

**答案：**

元表是控制另一个值特定行为的 table。常见元方法有 __index、__newindex、__add 和 __call。

对普通 table，读缺失键可触发 __index，给缺失键赋值可触发 __newindex；已有键通常直接读写。元方法内部可用 rawget／rawset 绕过相应拦截，避免递归。

### 5. Lua 如何实现面向对象？点号和冒号有什么区别？

**答案：**

用 table 保存对象状态，元表的 __index 指向共享方法表，可继续建立继承查找链。

```lua
local Actor = {}
Actor.__index = Actor
function Actor.new(hp)
    return setmetatable({hp = hp}, Actor)
end
function Actor:damage(amount)
    self.hp = self.hp - amount
end
local actor = Actor.new(100)
actor:damage(10) -- 等价于 actor.damage(actor, 10)
```

冒号语法隐式传 self；实例可变数据放在实例表中，避免所有对象共享同一份可变状态。

### 6. 闭包和 upvalue 是什么？

**答案：**

闭包是函数及其捕获环境。内部函数访问外部局部变量时，这些被捕获变量称为 upvalue，即使外层函数结束，也可被仍存活的闭包保留。

捕获通常引用变量，而非简单创建值快照。事件、定时器长期保存闭包时，可能连带保留对象，需要在业务结束时解除引用。

## 二、GC 与生命周期

### 7. Lua 的 GC 是什么机制？

**答案：**

GC 从根引用追踪可达对象，回收不可达对象；普通孤立循环引用可以回收。Lua 5.1 使用增量标记清扫，Lua 5.4 提供增量和分代模式，LuaJIT 的实现另行确认。

频繁分配、长寿命缓存和未清理回调都可能增加内存和回收压力。调整 GC 参数前先观察分配与存活量。

### 8. Lua 如何检查和处理内存泄漏？

**答案：**

反复执行进入／退出流程，对比对象数量与内存，检查全局表、模块缓存、事件、定时器、协程、闭包和 registry 的引用链。

移除本应结束的强引用，按生命周期注销回调并释放桥接资源。`collectgarbage("count")` 只是一个观测值，不能单凭未下降就认定所有内存泄漏。

### 9. 弱表有什么作用？

**答案：**

元表 `__mode` 可设 k、v 或 kv，表示弱键、弱值或两者。弱引用不会单独使相应可回收对象保持存活，适合辅助索引或缓存。

条目可能被回收，因此查询要处理缺失；弱表不能替代文件、句柄等外部资源的确定性释放。

### 10. Lua 与 C#／C++ 交互时如何管理对象生命周期？

**答案：**

桥接可用 userdata、registry 引用、托管委托和句柄关联对象。双方互相持有时，单侧 GC 不一定能判断完整业务关系。

明确订阅／解除、获取／释放和 Dispose 契约；退出时先停回调，再解除桥接引用。动态加载和异步返回还要检查接收对象是否仍有效。

## 三、常见函数与协程

### 11. Lua 字符串如何拼接？大量拼接怎么优化？

**答案：**

少量字符串使用 `..`。循环拼接长字符串可能反复复制，可将片段放入数组，再调用 table.concat。

```lua
local parts = {}
for i = 1, 100 do parts[#parts + 1] = tostring(i) end
local text = table.concat(parts, ",")
```

片段区间应连续且类型符合 concat 要求，同时关注最终结果的峰值内存。

### 12. table.sort 使用什么算法？比较函数怎么写？

**答案：**

标准 Lua 5.4 的实现属于快速排序体系，API 不保证稳定。比较器应表达严格的“排在前面”，例如 `a.score < b.score`，不能用 `<=`。

需要相等元素顺序可预测时，加入原始索引等第二关键字；排序作用于原序列，不返回新 table。

### 13. Lua 协程的原理、状态和使用方法是什么？

**答案：**

协程是协作式执行单元：resume 启动／恢复，yield 主动让出；暂停后保留状态，下次继续。状态包括 suspended、running、normal、dead。

```lua
local co = coroutine.create(function(x)
    local nextValue = coroutine.yield(x + 1)
    return nextValue * 2
end)
print(coroutine.resume(co, 10)) -- true, 11
print(coroutine.resume(co, 7))  -- true, 14
```

resume 首个返回值表示成功与否。协程不自动创建并行线程，同线程死循环仍会阻塞其他工作。

### 14. Lua 和 C 如何通过 C API 交互？

**答案：**

C API 使用虚拟栈传递参数和返回值。C 调 Lua 时加载代码、压入函数与参数、保护调用并取结果；Lua 调 C 时将 lua_CFunction 注册到环境，函数读取栈参数、压入结果并返回结果数量。

桥接需要保持栈平衡，检查参数和错误，明确 userdata 的所有权。Lua 错误传播与 C++ 清理机制的衔接也应正确处理。

参考：[Lua C API](https://www.lua.org/manual/5.4/manual.html#4)。相关所有权问题见第 8 题及 [C++ 专题](C++高频题整理.md)。

### 15. Lua 函数里一行报错，代码会继续执行吗？报错位置不确定时怎么定位？

**答案：**

不会继续执行同一条调用链：Lua 错误通过错误对象沿调用栈向上抛，中断当前函数及其所有调用者；未捕获时终止整个入口调用并回到宿主（C 侧 `lua_pcall` 返回错误）。它不像某些语言那样“跳过出错行继续往下跑”。

防御与定位：

- 保护调用：可能出错的边界（定时器回调、事件分发、单条业务消息处理）用 `pcall`／`xpcall` 包裹，`xpcall` 附带错误处理函数拿到 `debug.traceback`，防止一个 UI 回调炸掉整个分发循环。
- 定位不确定的报错：`debug.traceback` 给调用栈；`debug.getinfo`／钩子（`debug.sethook`）在计数或行事件上采集，配合日志把“哪个文件哪一行”打全；线上版本避免重钩子，用错误对象＋traceback 上报即可。
- 错误即值：`error()` 抛出的对象可以是 table（带错误码），业务层按错误对象分级处理，而不是全靠字符串匹配。

与宿主的边界：C++ RAII 与 `lua_pcall` 的长跳转交互（见第 14 题），错误处理函数里不要再抛错。

## 参考资料

- [Lua 5.4 手册](https://www.lua.org/manual/5.4/manual.html)
