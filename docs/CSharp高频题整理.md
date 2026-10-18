# C# 面试题与答案

> 共 136 题。按主题分类，答案直接放在题目后。

[返回题库总目录](面试题库总目录.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、基础与类型语义](#%E4%B8%80%E3%80%81%E5%9F%BA%E7%A1%80%E4%B8%8E%E7%B1%BB%E5%9E%8B%E8%AF%AD%E4%B9%89) | 1–22 |
| [二、面向对象与多态](#%E4%BA%8C%E3%80%81%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E4%B8%8E%E5%A4%9A%E6%80%81) | 23–32 |
| [三、泛型、反射与比较规则](#%E4%B8%89%E3%80%81%E6%B3%9B%E5%9E%8B%E3%80%81%E5%8F%8D%E5%B0%84%E4%B8%8E%E6%AF%94%E8%BE%83%E8%A7%84%E5%88%99) | 33–44 |
| [四、委托、闭包与事件](#%E5%9B%9B%E3%80%81%E5%A7%94%E6%89%98%E3%80%81%E9%97%AD%E5%8C%85%E4%B8%8E%E4%BA%8B%E4%BB%B6) | 45–54 |
| [五、内存管理与资源释放](#%E4%BA%94%E3%80%81%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B8%8E%E8%B5%84%E6%BA%90%E9%87%8A%E6%94%BE) | 55–77 |
| [六、集合与迭代器](#%E5%85%AD%E3%80%81%E9%9B%86%E5%90%88%E4%B8%8E%E8%BF%AD%E4%BB%A3%E5%99%A8) | 78–88 |
| [七、LINQ 查询与性能](#%E4%B8%83%E3%80%81LINQ%20%E6%9F%A5%E8%AF%A2%E4%B8%8E%E6%80%A7%E8%83%BD) | 89–108 |
| [八、异步与并发](#%E5%85%AB%E3%80%81%E5%BC%82%E6%AD%A5%E4%B8%8E%E5%B9%B6%E5%8F%91) | 109–129 |
| [九、异常处理](#%E4%B9%9D%E3%80%81%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86) | 130–132 |
| [十、设计与测试基础](#%E5%8D%81%E3%80%81%E8%AE%BE%E8%AE%A1%E4%B8%8E%E6%B5%8B%E8%AF%95%E5%9F%BA%E7%A1%80) | 133–134 |
| [十一、运行时与执行模型](#%E5%8D%81%E4%B8%80%E3%80%81%E8%BF%90%E8%A1%8C%E6%97%B6%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%A8%A1%E5%9E%8B) | 135–136 |

## 一、基础与类型语义

### 1. 值类型和引用类型有什么区别？

**答案：**

值类型通常直接存储数据，赋值时复制数据；引用类型的变量存储对象引用，赋值时复制引用，多个变量可能指向同一个对象。

- 值类型：`int`、`float`、`bool`、`struct`、`enum`。
- 引用类型：`class`、`string`、数组、委托。

注意：值类型不一定在栈上，引用类型对象也不一定始终在堆上，具体取决于运行时和使用场景。

### 2. `struct` 和 `class` 有什么区别？

**答案：**

- `struct` 是值类型，赋值时复制其值。
- `class` 是引用类型，赋值时复制引用。
- `struct` 不能继承其他结构体或普通类，但可以实现接口。
- `class` 支持类继承。
- 小型、轻量、具有值语义的数据适合使用 `struct`。

### 3. `ref`、`out`、`in` 有什么区别？

**答案：**

- `ref`：参数必须提前初始化，方法可以读取和修改它。
- `out`：调用前不要求初始化，方法必须在返回前赋值。
- `in`：以只读引用传递参数，避免大型结构体的复制。

```csharp
void A(ref int x) { x++; }
void B(out int x) { x = 10; }
void C(in decimal amount) { }
```

### 4. `const` 和 `readonly` 有什么区别？`readonly` 对象还能修改吗？

**答案：**

- `const`：编译期常量，声明时必须初始化；常量字段隐式为静态成员。
- 实例 `readonly` 字段：只能在声明处或所属类型的实例构造函数中赋值。
- `static readonly` 字段：只能在声明处或所属类型的静态构造函数中赋值。
- `readonly` 修饰引用类型字段时，限制的是引用的重新赋值，不保证对象内部状态不可变。

```csharp
class Settings
{
    public const int MaxCount = 100;
    private readonly List<int> values = new List<int>();

    public void Add()
    {
        values.Add(1);                  // 可以修改对象内容
        // values = new List<int>();   // 不能重新赋值该字段
    }
}
```

### 5. `string` 为什么不可变？如何选择 `string` 和 `StringBuilder`？

**答案：**

字符串创建后，不能直接修改其中的字符。看似修改字符串的操作，通常会创建新字符串或返回另一个字符串。

- **优点**：内容和哈希值稳定，便于共享，也更容易保证字符串内容的线程安全。
- **代价**：频繁拼接可能产生大量中间字符串，增加分配和 GC 压力。
- **选择**：少量拼接使用 `string`；循环中大量拼接可使用 `StringBuilder`，减少中间字符串。

字符串不可变，不代表保存字符串的变量不能重新赋值。

### 6. `==` 和 `Equals()` 有什么区别？

**答案：**

- `==` 是运算符，可以被重载；默认行为取决于类型。
- `Equals()` 是方法，可以被重写，用于定义相等性。
- 对于普通类，默认通常比较引用；对于 `string`，两者默认都比较字符串内容。

`ReferenceEquals(a, b)` 用于判断引用身份，不调用类型重载的 `==`。比较值类型时，传入 `object` 参数还可能发生装箱。

### 7. 什么是装箱和拆箱？为什么泛型可以减少装箱？

**答案：**

- **装箱**：将值类型转换为 `object` 或其实现的接口类型，通常需要创建装箱对象。
- **拆箱**：从装箱对象中取出对应的值类型，要求装箱类型与拆箱目标类型匹配。

```csharp
int x = 10;
object obj = x;                        // 装箱
int y = (int)obj;                      // 拆箱

List<int> numbers = new List<int>();
numbers.Add(x);                        // 存储 int 不需要装箱
```

泛型集合能够直接保存指定类型的数据，避免非泛型容器把值类型当作 `object` 存储时的装箱。但将泛型值传给 `object` 参数，或执行某些接口转换时，仍可能装箱。频繁装箱会增加 GC 压力。

拆箱取值不代表结果必然落在栈上；值也可以写入对象字段、数组元素或保存在寄存器中。值类型和引用类型的区别是值语义与引用语义，而不是固定存储在栈或堆。

### 8. `object`、`dynamic`、`var` 有什么区别？

**答案：**

| 写法 | 类型与绑定时机 | 主要特点 |
| --- | --- | --- |
| `object` | 静态类型为 `object` | 访问具体类型成员通常需要转换；接收值类型时可能装箱 |
| `var` | 编译期推断出明确的静态类型 | 不是独立的运行时类型，推断后不能随意改变变量类型 |
| `dynamic` | 许多成员绑定和检查推迟到运行时 | 通常依赖运行时绑定器，成员调用失败可能在运行时抛出异常 |

```csharp
var x = 10;       // x 的静态类型是 int
dynamic y = 10;
y = "hello";      // 允许
object z = 10;    // int 装箱为 object
```

### 9. `is`、`as` 和强制类型转换有什么区别？

**答案：**

- `is`：判断对象是否兼容某个类型。
- `as`：转换失败返回 `null`，适用于引用类型和可空值类型等支持的转换。
- 强制转换：转换失败时可能抛出异常。

```csharp
if (obj is Player player)
{
    player.Move();
}

Player player2 = obj as Player;
Player player3 = (Player)obj;
```

### 10. `readonly struct` 有什么作用？

**答案：**

声明只读结构体，编译器会限制实例状态被修改。若字段包含引用，引用指向的对象仍可能被修改，因此不代表深度不可变。它有助于表达值语义，并减少部分防御性复制，但具体优化效果取决于使用方式。

```csharp
readonly struct Position
{
    public readonly float X;
    public readonly float Y;

    public Position(float x, float y)
    {
        X = x;
        Y = y;
    }
}
```

### 11. 什么是防御性复制（Defensive Copy）？

**答案：**

当通过只读引用访问非只读成员时，编译器可能复制结构体，避免通过该成员修改原对象。

```csharp
struct Data
{
    public int Value;
    public void Print() => Console.WriteLine(Value);
}

readonly struct Wrapper
{
    public readonly Data Data;
}
```

如果通过 `Wrapper.Data` 调用非只读成员，某些情况下会产生防御性复制。对于频繁访问的大型结构体，应考虑使用 `readonly` 成员或只读结构体。

### 12. `ref return` 和 `ref local` 是什么？

**答案：**

允许通过引用返回或保存变量，而不是复制其值。

```csharp
static ref int GetRef(int[] array, int index)
{
    return ref array[index];
}

int[] values = { 1, 2, 3 };
ref int value = ref GetRef(values, 0);
value = 100; // values[0] 变为 100
```

引用不能指向生命周期不足的局部变量。`ref` 返回可以用于高性能数据访问，但必须遵守 C# 的引用安全规则。

### 13. in 参数适合哪些场景？

**答案：**

`in` 传递只读引用，可以避免大型结构体复制；但对小型结构体，引用传递本身有成本，而且通过只读引用调用可变结构体的非只读成员时，可能触发防御性复制。因此应根据结构体大小、访问方式和实际测量决定。

### 14. `record` 和普通 `class` 有什么区别？

**答案：**

`record` 默认强调值相等性，而普通类默认采用引用相等性。`record` 还支持便捷的 `with` 表达式和数据模型语法。

```csharp
record PlayerData(int HP, int Level);

var a = new PlayerData(100, 1);
var b = new PlayerData(100, 1);

Console.WriteLine(a == b); // True
```

对于 `record class`，这是基于成员的值相等性；对于 `record struct`，则是值类型语义。

### 15. C# 中的 `checked` 和 `unchecked` 有什么区别？

**答案：**

它们控制整数算术溢出的检查行为。

```csharp
int x = int.MaxValue;

int a = unchecked(x + 1); // 通常得到 int.MinValue
int b = checked(x + 1);   // 抛出 OverflowException
```

默认行为取决于编译设置和上下文；浮点数运算不受这种整数溢出检查机制的控制。

### 16. C# 默认按值传参，为什么方法仍能修改引用对象？

**答案：**

默认传递的是参数值的副本。引用类型参数复制的是引用，所以可以修改同一个对象的成员；将参数重新指向另一个对象，却不会改变调用方变量。`ref` 则允许方法直接修改调用方变量。

```csharp
static void Change(List<int> items)
{
    items.Add(1);                  // 调用方能看到元素变化
    items = new List<int>();      // 只改变方法内部的引用副本
}
```

### 17. `int?` 与 `string?` 有什么区别？

**答案：**

- `int?` 是 `Nullable<int>`，属于可空值类型，可以表示整数或无值；访问无值对象的 `Value` 会抛出异常。
- 启用可空引用类型分析后，`string?` 表达“可能为 null”的引用契约，主要用于静态分析；它不是独立于 `string` 的运行时类型。
- 非可空引用标注不会自动插入所有运行时检查，公开入口仍可能需要参数校验。

```csharp
int? score = null;
int actualScore = score ?? 0;
```

### 18. `?.`、`??`、`??=` 和空值宽容运算符 `!` 有什么区别？

**答案：**

- `?.`：接收者为 null 时跳过成员访问并产生 null；不会吞掉成员内部的异常。
- `??`：左侧为 null 时求值并返回右侧。
- `??=`：左侧为 null 时才赋值。
- 后缀 `!`：告知静态分析忽略此处的可空警告，不执行运行时检查。

```csharp
string name = user?.Name ?? "Guest";
cache ??= new Dictionary<string, int>();
// user!.Name 中的 ! 不能防止 user 实际为 null
```

### 19. `float`、`double` 和 `decimal` 如何选择？

**答案：**

`float`、`double` 使用二进制浮点表示，适合科学计算等场景；`double` 通常具有更高精度。`decimal` 使用十进制缩放表示，适合金额等需要准确表示常见十进制小数的场景，但范围、速度与二进制浮点不同。

二进制浮点不能准确表示许多十进制小数。比较计算结果时，应根据业务使用绝对或相对误差，不能把固定误差阈值当作所有数量级的通用答案。`decimal` 也有有限精度，例如 `1m / 3m` 仍需舍入。

### 20. 浅拷贝、深拷贝与 `record` 的 `with` 有什么区别？

**答案：**

浅拷贝复制当前对象的字段值；引用字段复制引用，嵌套对象仍然共享。深拷贝根据业务需要递归复制相关对象，并处理循环引用和共享关系。

`MemberwiseClone()` 是浅拷贝；`record class` 的 `with` 默认也不会递归复制所有成员。对引用类型元素执行 `ToList()` 或 `ToArray()` 同样只是创建新容器，元素仍引用原对象。

### 21. 下面的字符串代码会产生哪些对象？

**答案：**

```csharp
string a = new string(new[] { 'a', 'b', 'c' });
a = (a.ToUpperInvariant() + "123").Substring(0, 2);
```

按常见实现中的显式分配分析：char 数组、初始字符串、大写字符串、拼接字符串和子串，共五项新建对象候选；字符串字面量通常按驻留常量另计。

原因是 string 不可变，修改内容通常生成新字符串。具体分配可受运行时优化或结果复用影响；面试时先明确是否统计字面量及最终结果。

### 22. C# 和 C++ 有什么区别？

**答案：**

| 方面 | C# | C++ |
| --- | --- | --- |
| 执行环境 | 通常依赖托管运行时，可使用 JIT／AOT | 通常生成原生程序 |
| 内存管理 | GC 管托管内存，Dispose 管资源 | RAII、值语义、智能指针 |
| 类型机制 | class 引用语义、struct 值语义、泛型 | class／struct 均可值语义、模板 |
| 继承 | 类单继承，可实现多个接口 | 支持多继承 |
| 底层控制 | 可通过互操作和 unsafe 访问底层 | 布局、所有权和原生接口控制更直接 |

两者都支持面向对象等风格，实际性能要结合算法、数据布局与运行方式。

## 二、面向对象与多态

### 23. 面向对象的三大特性是什么？

**答案：**

- 封装：隐藏内部实现，通过公开接口访问数据。
- 继承：子类复用并扩展父类行为。
- 多态：同一接口调用不同对象时，可以执行不同实现。

### 24. 抽象类和接口有什么区别？

**答案：**

- 抽象类可以包含字段、构造函数、状态和已实现的方法。
- 接口主要定义类型必须提供的契约，也支持默认接口实现等现代 C# 特性。
- 类只能直接继承一个类，但可以实现多个接口。

抽象类适合共享状态和基础实现；接口适合定义 `IDisposable`、`IComparable<T>` 等可由不同类型实现的能力。

### 25. `virtual`、`override`、`new` 有什么区别？

**答案：**

- `virtual`：允许子类重写方法。
- `override`：重写基类的虚方法，实现运行时多态。
- `new`：隐藏基类成员，不是重写；通过基类引用调用时，仍可能执行基类版本。

```csharp
class A
{
    public virtual void Run() { }
}

class B : A
{
    public override void Run() { }
}
```

### 26. 什么是多态？虚方法如何实现运行时分派？

**答案：**

通过基类或接口引用调用方法时，根据对象的实际运行时类型执行对应实现，这就是运行时多态。

- **虚方法调用**：运行时通过类型信息和虚方法分派机制确定目标方法。
- **接口调用**：通过接口分派机制找到实际实现。
- **优化**：在目标类型可确定时，JIT 可能将虚调用优化成直接调用，即去虚拟化。

可以借助 C++ 虚函数表理解这一机制，但不能认为所有 .NET 运行时都采用完全相同的底层实现。

### 27. 什么是协变返回类型？

**答案：**

重写虚方法时，允许返回比基类声明更具体的引用类型。

```csharp
class Animal { }

class Dog : Animal { }

class Factory
{
    public virtual Animal Create() => new Animal();
}

class DogFactory : Factory
{
    public override Dog Create() => new Dog();
}
```

这使子类可以提供更具体的返回类型，同时保留多态调用能力。

### 28. 重载与重写有什么区别？

**答案：**

**重载**是在同一方法名下定义不同参数列表，普通静态绑定调用依据调用处类型选择签名；不能仅凭返回类型重载。**重写**使用 `override` 替换继承的虚成员实现，由运行时实际对象类型参与分派。

面试时要区分“先选择调用哪个签名”和“该虚成员最终由哪个类型实现”。`dynamic` 调用的成员绑定时机会不同。

### 29. `static` 类和单例有什么区别？

**答案：**

静态类不能实例化，通常承载无实例状态的工具方法。单例是控制某个类实例数量的设计方式，实例仍可实现接口、作为依赖传递。

静态字段只说明状态属于类型，不说明访问线程安全。单例还需要考虑创建时机、并发、生命周期和测试替换；“全局可访问”本身并不是单例的充分理由。

### 30. `sealed` 有什么作用？

**答案：**

`sealed class` 阻止进一步继承；`sealed override` 阻止子类继续重写该成员，但不禁止继承包含它的类。

它能明确扩展边界，也可能为去虚拟化提供条件，但不能据此保证每次调用都会更快。`sealed` 与不可变、线程安全是不同概念。

### 31. 扩展方法是什么？如何调用？

**答案：**

传统扩展方法声明在静态类中，方法本身为静态方法，第一个参数用 `this` 指定接收者类型。它提供类似实例调用的语法，不会修改原类型，也不能访问其私有成员。

```csharp
public static class TextExtensions
{
    public static bool IsBlank(this string value)
        => string.IsNullOrWhiteSpace(value);
}
```

存在适用的实例方法时，实例方法优先。扩展方法按调用处可见类型进行选择，不是虚方法重写。

### 32. `public`、`private`、`protected`、`internal` 及其组合有什么区别？

**答案：**

| 修饰符 | 访问范围 |
| --- | --- |
| public | 在包含类型本身可访问的前提下公开 |
| private | 限定在所属类型的访问上下文 |
| protected | 所属类型及派生类型的受保护访问规则 |
| internal | 同一程序集，不是笼统的同一文件夹或项目 |
| protected internal | 同一程序集，或派生类型访问 |
| private protected | 同一程序集内的所属类型及派生类型访问 |

还要考虑外层类型可访问性；不能通过 public 成员暴露更不可访问的类型。`override` 是重写，`new` 隐藏成员，不应与访问修饰符混为一类。

## 三、泛型、反射与比较规则

### 33. 什么是泛型？有什么好处？

**答案：**

泛型允许类型或方法使用类型参数，例如 `List<T>`。

好处：

- 编译期类型安全。
- 减少显式类型转换。
- 值类型使用泛型集合时通常可以避免装箱。

```csharp
List<int> numbers = new();
```

### 34. 什么是 `new()` 泛型约束？有什么限制？

**答案：**

```csharp
class Factory<T> where T : new()
{
    public T Create() => new T();
}
```

它要求 `T` 是非抽象类型，并具有公共无参构造函数。对于值类型，默认构造同样满足该约束。

不能使用它直接调用任意参数的构造函数。

### 35. `where T : class`、`where T : struct`、`where T : unmanaged` 有什么区别？

**答案：**

- `class`：要求 `T` 为引用类型。
- `struct`：要求 `T` 为非可空值类型。
- `unmanaged`：要求 `T` 为非托管类型，即不包含引用类型字段的值类型。

`unmanaged` 常用于直接操作内存的场景；它本身不保证特定的字段布局、对齐方式或跨语言数据兼容性。

### 36. 什么是协变和逆变？为什么 `List<T>` 不支持协变？

**答案：**

协变和逆变描述泛型接口、委托在类型参数不同的情况下允许的类型转换，相关变体转换适用于引用类型参数。

- **协变 `out T`**：允许把输出更具体类型的对象，当作输出更一般类型的对象使用。
- **逆变 `in T`**：允许把能够接收更一般类型的对象，当作只需接收更具体类型的对象使用。

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // 协变

Action<object> printObject = value => Console.WriteLine(value);
Action<string> printString = printObject; // 逆变
```

`IEnumerable<out T>` 输出 `T`，可以协变；`List<T>` 既读取又写入 `T`。如果允许 `List<string>` 转成 `List<object>`，就可能向字符串列表写入整数，破坏类型安全，因此不允许这种转换。

### 37. 泛型代码在运行时如何处理？

**答案：**

CLR 的泛型实现会根据类型参数和运行时策略生成或共享代码。常见 .NET CLR 会对不同值类型实例化采用不同的专用代码，而引用类型实例化通常可以共享部分泛型代码。

这与 C++ 模板常见的编译期实例化机制不同，但具体实现细节取决于运行时。

### 38. 泛型约束有哪些？有什么作用？

**答案：**

泛型约束声明类型参数必须具备的能力，使泛型代码可以安全地使用相应成员和操作。

常见有 class／struct／unmanaged、基类／接口约束、new() 等。例如 `where T : IDisposable` 允许调用 Dispose，`where T : new()` 允许创建 T 实例。

约束可以表达接口契约、减少不必要的运行时检查，但不代表任意约束都会直接带来性能提升。

### 39. 什么是表达式树？它与 Lambda 有什么区别？

**答案：**

普通 Lambda 可以编译为委托；表达式树则把 Lambda 表示为可检查、可分析的代码结构。

```csharp
Expression<Func<int, int>> expr = x => x + 1;
Func<int, int> func = expr.Compile();
```

表达式树可用于动态查询、规则解析和代码生成，但构建、分析或编译表达式树本身也可能有明显开销。

### 40. 什么是反射？原理、用途和性能优化方法是什么？

**答案：**

反射通过运行时元数据获取类型、字段、属性和方法信息，并可动态创建对象或调用成员。

常用于序列化、编辑器工具、插件注册和依赖注入。常见入口是 Type、FieldInfo、PropertyInfo、MethodInfo。

查找、动态调用和装箱可能增加成本；可缓存元数据、生成工厂／委托，避免在热点重复查找。裁剪／AOT 环境中还需保留被动态访问的类型和成员。

### 41. Activator.CreateInstance 如何创建对象？

**答案：**

它根据运行时 Type 或泛型参数创建实例，适合插件、反射工厂和不知道静态具体类型的场景。

普通已知类型优先直接构造。高频动态创建可缓存工厂委托，减少重复查找；目标类型还需有可用的构造方式，并处理抽象类型、访问权限和参数不匹配等失败。

### 42. `IEquatable<T>` 与 `IEqualityComparer<T>` 有什么区别？

**答案：**

`IEquatable<T>` 由类型自身定义强类型相等性；`IEqualityComparer<T>` 是外部比较策略，同时提供 `Equals` 和 `GetHashCode`。

同一种数据可能有多种比较规则，例如字符串区分或忽略大小写。集合可注入比较器，不必修改类型本身。`EqualityComparer<T>.Default` 会利用类型提供的相等性能力；两种方式都必须满足“相等对象具有相同哈希值”。

### 43. `IComparable<T>` 与 `IComparer<T>` 有什么区别？

**答案：**

二者描述排序关系，返回值小于零、等于零、大于零分别表示前者小于、等于、大于后者。

- `IComparable<T>`：类型提供默认比较规则。
- `IComparer<T>`：外部提供不同排序策略，例如按价格或按名称排序。

排序相等与业务上的对象相等不一定相同。例如按年龄比较时，两个人年龄相同，不代表是同一个人。

### 44. 序列化和反序列化是什么？

**答案：**

序列化按约定把数据编码为可存储或传输的表示，反序列化再恢复对应数据。格式、字段规则、类型信息、循环引用和版本兼容由方案决定。

线程、文件句柄、委托、原生指针等运行时状态通常不能简单照搬；应保存稳定 ID 或重建所需信息。处理外部数据时校验长度、类型与范围，避免让不可信内容任意选择实例化类型。

## 四、委托、闭包与事件

### 45. 什么是委托和事件？

**答案：**

- 委托：类型安全的方法引用，可以保存一个或多个符合签名的方法。
- 事件：基于委托的发布订阅机制，限制外部代码直接触发事件。

```csharp
public event Action<int> OnHealthChanged;

void TakeDamage(int damage)
{
    OnHealthChanged?.Invoke(damage);
}
```

事件常用于 UI 更新、战斗状态通知和模块间解耦。订阅者不再需要时应及时取消订阅。

### 46. `Action`、`Func` 和自定义委托有什么区别？

**答案：**

- `Action<T>`：没有返回值的委托。
- `Func<T, TResult>`：具有返回值的委托，最后一个泛型参数表示返回类型。
- 自定义委托：可以定义更具业务语义的签名，也可以在特定场景下提供更清晰的 API。

```csharp
Action<int> action = x => Console.WriteLine(x);
Func<int, int> func = x => x * 2;
```

### 47. 多播委托的返回值有什么特点？

**答案：**

多播委托可以依次调用多个方法。如果有返回值，普通同步调用最终只返回最后一个被调用方法的返回值；前面方法的返回值会被忽略。

```csharp
Func<int> f = () => 1;
f += () => 2;

int result = f(); // result 为 2
```

如果某个方法抛出异常且异常没有被内部处理，后续委托通常不会继续执行。

### 48. 闭包是什么？为什么可能造成内存问题？

**答案：**

闭包是能够捕获外部变量的函数。编译器通常会将被捕获变量放入生成的对象中，使其生命周期超出原方法调用。

频繁创建捕获局部变量的 Lambda，可能产生额外分配；如果长期保存委托，还可能延长被捕获对象的生命周期。

### 49. Lambda 的捕获与内存分配有什么关系？

**答案：**

不一定。

- 捕获外部局部变量的 Lambda，通常需要生成闭包对象。
- 不捕获外部变量的 Lambda，可能被缓存或转换成静态方法引用。
- 最终是否分配，取决于编译器、运行时、委托是否缓存及具体使用方式。

性能优化应以分析工具和目标 .NET 运行时的实际结果为准。

### 50. 闭包捕获的是变量还是变量的值？

**答案：**

通常捕获的是变量本身的存储位置，而不是创建 Lambda 时变量的快照。

```csharp
var actions = new List<Action>();

for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i));
```

上述代码只是创建委托。若在循环结束后依次调用这些委托，会输出 `3、3、3`。如果想输出 `0、1、2`，应在循环内部创建局部副本：

```csharp
for (int i = 0; i < 3; i++)
{
    int index = i;
    actions.Add(() => Console.WriteLine(index));
}
```

### 51. 为什么事件订阅可能造成内存泄漏？如何避免？

**答案：**

发布者通过委托持有订阅者的引用。如果发布者寿命更长，订阅者即使在业务上不再使用，也可能仍然可达，无法被 GC 回收。

- 在订阅结束时显式取消订阅，也可用实现 `IDisposable` 的订阅句柄管理生命周期。
- 静态事件、长期缓存的发布者尤其容易保留对象。
- 需要取消 Lambda 订阅时，保存原委托，不要重新写一个 Lambda 代替它。

```csharp
EventHandler handler = (sender, args) => Refresh();
publisher.Changed += handler;
// 订阅生命周期结束
publisher.Changed -= handler;
```

### 52. 如何设计一个避免重复分配的高频事件系统？

**答案：**

- 预先创建和复用事件对象或参数缓冲区。
- 避免每次派发都创建新的委托或闭包。
- 为订阅列表预分配容量。
- 明确订阅和取消订阅的生命周期。
- 若使用对象池，归还时重置事件状态。
- 对多线程生产、主线程消费的场景，使用合适的线程安全队列。

但不能为了零分配而牺牲可维护性；应优先优化 性能分析 证明存在问题的路径。

### 53. event 和公开委托字段有什么区别？

**答案：**

普通公开委托字段允许外部重新赋值、清空甚至直接调用。`event` 将外部操作约束为订阅和取消订阅，保护发布者对通知流程的控制。

事件可使用自定义 `add`／`remove` 访问器管理订阅，因此不能假定所有事件都只是一个可直接访问的委托字段。普通事件也不会自动让业务回调变得线程安全。

### 54. `for` 与 `foreach` 中捕获循环变量有什么区别？

**答案：**

现代 C# 中，`foreach` 的迭代变量在每次迭代有独立的捕获语义；`for` 初始化部分声明的变量则通常是各个闭包共享的同一个变量。

```csharp
var callbacks = new List<Action>();
foreach (int number in new[] { 0, 1, 2 })
    callbacks.Add(() => Console.WriteLine(number));

foreach (Action callback in callbacks)
    callback(); // 依次输出 0、1、2
```

独立捕获引用变量，也不代表对其指向的对象进行深拷贝。

## 五、内存管理与资源释放

### 55. C# 如何管理托管内存？什么是 GC Roots？

**答案：**

托管对象由运行时的垃圾回收器管理。GC 从 **GC Roots** 出发，沿引用关系追踪可达对象；不可达对象才有资格被回收。

常见根引用包括：

- 活动线程栈上的引用。
- 静态字段中的引用。
- 运行时维护的其他根引用。

“业务上不再使用”不等于“GC 不可达”。如果对象仍被静态集合、缓存或事件委托引用，就可能继续存活。

### 56. 什么情况下会触发 GC？

**答案：**

当托管堆分配压力达到一定程度、运行时判断需要回收，或者代码显式调用 `GC.Collect()` 时，都可能触发 GC。

应关注分配速率、GC 回收频率和暂停时间，不能认为每次分配都会立刻触发 GC。

### 57. 什么是 GC Alloc？如何减少？

**答案：**

GC Alloc 指当前执行过程中产生的托管堆分配。分配本身不等于立即发生 GC，但持续分配会增加回收压力。

常见优化：

- 避免在高频调用的方法 中创建临时对象。
- 复用集合和缓存。
- 避免不必要的装箱。
- 使用对象池管理高频创建和销毁的对象。
- 减少字符串拼接和闭包分配。
- 使用 .NET 内存分析工具查看分配量和分配调用栈。

### 58. 什么是托管堆和非托管内存？

**答案：**

- 托管堆：由 CLR GC 管理的对象内存。
- 非托管内存：例如原生代码通过 `malloc` 分配的内存，通常需要显式释放或通过资源封装管理。

托管包装对象可能持有文件句柄或原生内存。包装对象被 GC 回收，不等于这些资源能在业务需要的时间及时释放；应明确资源所有权，并使用 `Dispose()` 等机制管理释放。

### 59. `IDisposable` 和析构函数有什么区别？

**答案：**

- `Dispose()`：用于显式释放资源，调用时机可控。
- C# 析构函数：由 CLR 作为终结器执行，时机不确定，不能保证立即执行。

```csharp
using (var stream = new FileStream(path, FileMode.Open))
{
    // 使用文件
} // 自动调用 Dispose()
```

实现 `IDisposable` 可以确定性释放文件句柄、原生资源等；复杂资源类型通常还需正确实现释放模式。

### 60. 什么是弱引用 `WeakReference`？

**答案：**

弱引用允许引用对象而不阻止 GC 回收该对象。适用于缓存等场景，但使用前必须判断目标对象是否仍然存活。

### 61. GC 为什么采用分代回收？

**答案：**

因为大量对象生命周期较短。如果每次回收都扫描整个堆，成本会很高。

.NET GC 将对象划分为不同代，优先回收较年轻的对象；存活时间较长的对象会晋升到更高代。这样可以利用对象生命周期的统计规律减少平均回收成本。

这是典型 .NET 分代 GC 的设计思路，具体回收策略和配置以目标运行时为准。

典型 .NET GC 使用 0、1、2 代，普通新小对象通常进入 0 代，存活对象可晋升；回收某一代会同时覆盖更年轻代。触发条件包含分配预算、内存压力等，不只是某一代“装满”。可达性标记后，运行时可采取清扫、压缩等策略，不是每次回收都移动全部对象。

### 62. 为什么频繁创建小对象也会影响性能？

**答案：**

小对象持续增加托管分配量，会消耗分配带宽并增加 GC 工作，影响吞吐量和响应延迟。

例如，每次操作分配 1 KiB、每秒执行 60 次，一分钟的累计分配量为：

`1 KiB × 60 × 60 = 3600 KiB ≈ 3.52 MiB`

累计分配量不等于同一时刻的存活内存。应减少热点中不必要的分配，并测量回收频率、暂停时间和实际吞吐量。

### 63. C# 中的对象在内存中通常包含什么？

**答案：**

托管对象通常包含运行时需要的对象头信息和实例数据。对象头可能涉及同步相关信息、类型信息等，具体布局由 CLR 实现决定，不属于 C# 语言保证的固定内存布局。

值类型则可能内嵌在对象、数组或栈帧中，也可能被装箱，不能简单认为值类型永远位于栈上。

### 64. struct 作为局部变量、字段和数组元素时怎样存储？

**答案：**

结构体字段通常直接内嵌在所属对象的内存布局中。

```csharp
struct Vector2
{
    public float X, Y;
}

class Entity
{
    public Vector2 Position;
}
```

创建 `Entity` 时，`Position` 通常是对象内部的一部分，不需要再为 `Vector2` 单独分配一个堆对象。

### 65. `GCHandle` 有什么作用？

**答案：**

`GCHandle` 允许托管代码显式控制对象的 GC 可达性或固定状态，常用于与原生代码交互。

常见类型：

- `Normal`：使对象保持可达。
- `Pinned`：固定对象，防止其被 GC 移动。
- `Weak`：弱引用形式，不阻止对象回收。

使用后必须适时调用 `Free()`，否则可能导致对象长期存活或固定内存，影响 GC。

### 66. 什么是 LOH？为什么大对象分配需要关注？

**答案：**

LOH 是 Large Object Heap（大对象堆）。在常见的 .NET GC 实现中，大约 85,000 字节及以上的对象通常会进入 LOH。

大对象分配和回收可能产生明显的内存管理成本，并增加碎片风险。具体阈值与行为应以目标运行时实现为准。

LOH 的阈值和整理策略属于运行时实现细节，不是 C# 语言规范的保证。

典型 .NET LOH 与第 2 代一起回收，通常采用清扫而非每次压缩；支持相应配置的运行时可请求压缩。不要把默认策略说成“永远不压缩”，也不要直接把 .NET 分代与 LOH 规则套到所有 Unity 运行时。

### 67. 什么是内存碎片？

**答案：**

内存碎片是指空闲内存分散在多个不连续区域，导致虽然总空闲量足够，却难以满足某些连续内存分配需求。

对象移动和堆压缩可以缓解部分碎片，但固定对象、原生堆分配和图形资源管理都有各自的约束。

### 68. GC 为什么需要写屏障（Write Barrier）？

**答案：**

分代 GC 需要知道老年代对象是否引用了新生代对象。如果程序修改对象引用时完全不记录这种变化，回收年轻代时可能遗漏仍然存活的对象。

写屏障会在引用写入时记录必要的信息，帮助 GC 正确追踪对象关系。

写屏障并非只服务于分代 GC；增量或并发标记也可能利用它记录引用变化。具体用途取决于 GC 实现。

### 69. 什么是逃逸分析？有什么优化作用？

**答案：**

逃逸分析判断对象或引用是否离开当前方法、作用域或线程。例如返回对象、保存到全局字段都可能让它逃逸。

若能够证明不逃逸，运行时可能消除分配、将数据拆成局部值，或采用其他更轻量存储。具体优化依运行时实现，C# 语言本身不保证所有不逃逸对象都放到栈上。

### 70. Span<T> 是什么？有什么作用？

**答案：**

`Span<T>` 是一种表示连续内存区域的类型，可以引用数组、栈内存或其他支持的连续内存，而不必为切片创建新的数组。

```csharp
int[] data = { 1, 2, 3, 4 };
Span<int> part = data.AsSpan(1, 2);

part[0] = 10; // 修改原数组中的 data[1]
```

它本身不需要为切片分配新数组。`Span<T>` 是 `ref struct`，不能像普通对象一样任意存入堆对象或跨越 `await` 等受限边界。

### 71. `stackalloc` 是什么？有什么风险？

**答案：**

`stackalloc` 在栈上分配一段连续内存，适合生命周期短、大小可控的临时缓冲区。

```csharp
Span<int> buffer = stackalloc int[16];
```

风险是栈空间有限，过大的分配或递归中的大量栈分配可能导致栈溢出。此外，它只适用于支持该特性的运行环境和语境，不能不加判断地用于大缓冲区。

### 72. `fixed` 关键字的作用是什么？

**答案：**

`fixed` 可以在指定范围内固定某些托管对象的位置，避免其在这段时间内被 GC 移动，从而允许安全地获取相关指针。

```csharp
unsafe
{
    int[] data = { 1, 2, 3 };

    fixed (int* p = data)
    {
        // 在此范围内访问固定后的内存
    }
}
```

固定时间过长会限制 GC 的内存整理效率。涉及原生代码互操作时，必须明确指针有效期与内存所有权。

### 73. `ArrayPool<T>` 有什么用？

**答案：**

它通过复用数组减少频繁分配大数组带来的 GC 压力，适合临时缓冲区。

```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096);

try
{
    // 使用 buffer，实际长度可能大于 4096
}
finally
{
    pool.Return(buffer);
}
```

归还后不能继续使用该数组。若缓冲区存放敏感数据，应考虑清零；若 `T` 包含引用，也应考虑归还时清除引用。

### 74. `using` 语句、`using` 声明和 `await using` 有什么区别？

**答案：**

- `using (...) { ... }`：退出语句块时释放资源。
- `using var resource = ...;`：在所属作用域结束时释放资源。
- `await using`：等待异步释放流程，通常用于实现 `IAsyncDisposable` 的资源。

同步 `using` 的资源清理语义类似 `try/finally`，即使作用域因异常退出也会尝试释放。它只管理释放时机，不代表操作本身线程安全，也不会自动把同步资源转换为异步资源。

### 75. GC 和 Dispose 分别负责什么？什么时候调用 GC.Collect？

**答案：**

GC 根据可达性回收托管对象占用的内存；Dispose 用于及时释放文件、连接和原生句柄等资源，通常配合 using。

GC.Collect 请求主动回收，但不会让仍有引用的对象变成不可达，也不能替代 Dispose。通常让运行时安排回收；在明确的阶段切换或诊断场景中，确认收益后再主动使用。

### 76. unsafe 和 P/Invoke 有什么作用？

**答案：**

unsafe 允许在相应上下文使用指针等受限制操作，适合部分底层互操作和内存处理，但不会自动关闭 GC，也不保证比安全代码更快。

普通 P/Invoke 通过合适的参数封送可无需 unsafe；需要直接操作地址时才考虑。应管理指针有效期、越界、固定对象与资源释放；能用 Span、标准库或可靠封装表达时，先评估这些方式。

### 77. 调用栈与托管堆有什么区别？

**答案：**

调用栈参与保存调用帧、返回信息及部分局部数据，作用域退出后相关栈空间可复用；托管堆中的对象通常根据可达性由 GC 管理，生命周期不必等于创建方法的调用期。

值类型可能内嵌于堆对象或数组，也可能装箱；局部变量还可能被提升、放入寄存器或被优化掉。因此不能以“值类型全在栈、引用类型全在堆”判断性能，也不能将原生堆的手动释放规则直接套用到托管堆。

## 六、集合与迭代器

### 78. `Array` 和 `List<T>` 有什么区别？`List<T>` 如何扩容？

**答案：**

| 对比项 | `Array` | `List<T>` |
| --- | --- | --- |
| 长度 | 创建后固定 | 元素数量可变 |
| 存储 | 数组元素槽位连续排列 | 内部使用数组存储 |
| 增长 | 需要自行创建新数组并复制 | 容量不足时创建更大的内部数组并复制 |
| 适用场景 | 数量固定的数据 | 需要动态增删的数据 |

`List<T>.Count` 是当前元素数量，`Capacity` 是内部数组容量。已知规模时可预先设置容量，减少扩容次数。

扩容策略取决于运行时实现，常见实现按比例增长。单次扩容复制通常为 `O(n)`，尾部添加的均摊复杂度为 `O(1)`。对于引用类型元素，连续存储的是引用，元素对象本身不必连续。

### 79. `Dictionary<TKey, TValue>` 的底层原理是什么？如何处理哈希冲突？

**答案：**

字典基于哈希表：先计算键的哈希值定位桶，再检查哈希值和键的相等性以找到实际条目。

- 常见 .NET 实现通过桶索引和条目链式结构处理冲突；同一个桶可能关联多个条目。
- 平均查找、插入和删除复杂度为 `O(1)`；冲突严重时性能可能退化。
- 自定义键类型需要正确实现 `Equals()` 和 `GetHashCode()`：相等的键必须具有相同的哈希值。
- 具体内部结构以目标 .NET 版本实现为准。

### 80. `List<T>` 和 `LinkedList<T>` 如何选择？

**答案：**

- `List<T>`：连续数组存储，遍历和随机访问性能好，适合大多数业务数据。
- `LinkedList<T>`：链表存储，已持有节点时插入和删除为 `O(1)`，但查找节点为 `O(n)`，且节点有额外内存开销。

不能简单认为链表插入删除快就更适合实际业务。

### 81. `IEnumerable<T>` 和 `IList<T>` 有什么区别？

**答案：**

- `IEnumerable<T>`：提供枚举能力。
- `IList<T>`：提供索引访问、插入、删除和修改等列表操作。

面向 `IEnumerable<T>` 编程可以降低接口依赖，但它不保证集合可以重复遍历、随机访问或修改。

### 82. 遍历 List 时如何安全删除元素？

**答案：**

使用 `foreach` 遍历集合时，如果集合发生结构性修改，枚举器会检测到版本变化并抛出异常。

```csharp
foreach (var item in list)
{
    if (item < 0)
        list.Remove(item); // 可能抛异常
}
```

可改为倒序遍历：

```csharp
for (int i = list.Count - 1; i >= 0; i--)
{
    if (list[i] < 0)
        list.RemoveAt(i);
}
```

### 83. `yield return` 的底层原理是什么？

**答案：**

编译器通常会将包含 `yield return` 的迭代器转换为状态机对象，保存当前执行位置和局部状态。

每次调用 `MoveNext()` 都会恢复执行，直到下一个 `yield return` 或结束。因此，迭代器通常可以延迟计算，但可能产生状态机对象和其他分配。

### 84. `HashSet<T>` 与 `List<T>` 有什么区别？

**答案：**

`HashSet<T>` 按相等性保存唯一元素，适合去重和频繁成员检测；`List<T>` 保留重复元素，提供按索引访问和列表顺序。

哈希分布合理时，`HashSet<T>.Contains()` 平均为 `O(1)`；无索引辅助的 `List<T>.Contains()` 为 `O(n)`。哈希集合有额外存储和哈希计算成本，小规模数据仍应按需求选择，也不应依赖哈希集合的枚举顺序。

### 85. 为什么不应修改字典键中参与哈希或相等性计算的字段？

**答案：**

插入时，字典根据当时的哈希值将键放入对应桶。若之后修改参与比较的字段，再查找时可能定位到另一个桶，导致同一个键对象也查不到。

键应优先使用不可变数据。需要修改键时，先按原状态删除，再用新键插入。`readonly` 引用字段不能保证键对象内部不可变。

### 86. IReadOnlyList 与不可变集合有什么区别？

**答案：**

IReadOnlyList 提供只读访问接口，调用方不能通过它增删元素，但底层集合仍可能被其他引用修改。

不可变集合保证集合结构创建后不被原地修改；若元素是可变引用对象，元素内部仍可能变化。需要稳定数据时，应区分只读接口、结构不可变和深度不可变。

### 87. IEnumerable、IEnumerator 和 foreach 的原理是什么？

**答案：**

`IEnumerable<T>` 提供枚举器；`IEnumerator<T>` 保存一次遍历的位置，通过 `MoveNext()` 和 `Current` 读取元素，并通常需要释放。

`foreach` 可使用枚举模式，不要求所有类型都以同一种方式实现接口。遍历数组或直接遍历 `List<T>` 可以避免某些分配；把结构体枚举器作为接口使用时可能装箱。分配情况取决于集合静态类型和实现，不能认为所有 `foreach` 都分配。

### 88. `ArrayList` 与 `List<T>` 有什么区别？

**答案：**

`ArrayList` 以 object 保存元素，可混入不同类型，取出时通常需要转换，值类型可能装箱。`List<T>` 固定元素类型，提供静态类型检查，并能避免把值类型作为 object 保存导致的装箱。

泛型不保证程序绝无装箱：若把元素转为 object 或走某些接口路径，仍可能产生。新业务一般优先使用合适的泛型集合。

## 七、LINQ 查询与性能

### 89. 什么是 LINQ？查询语法和方法语法有什么关系？

**答案：**

LINQ 是语言集成查询，用统一方式表达筛选、投影、排序、分组和聚合。常用操作符位于 `System.Linq`。

```csharp
int[] numbers = { 1, 2, 3, 4 };
var query1 = from n in numbers where n % 2 == 0 select n * 10;
var query2 = numbers.Where(n => n % 2 == 0).Select(n => n * 10);
// 枚举时二者都得到 20、40
```

查询语法会转换成对应方法调用。`Count()`、`ToList()` 等通常直接用方法语法。以下题目若未说明，默认讨论 LINQ to Objects。

### 90. 什么是延迟执行？修改数据源后查询结果会改变吗？

**答案：**

`Where`、`Select` 等通常只在枚举时读取数据，查询变量保存的是查询逻辑，不是结果快照。数据源或捕获的变量在枚举前发生变化，可能改变结果。

```csharp
var numbers = new List<int> { 1, 2, 3 };
int limit = 1;
var query = numbers.Where(n => n > limit);
limit = 2;
numbers.Add(4);
Console.WriteLine(string.Join(",", query)); // 3,4
```

`ToList()`、`ToArray()` 会在调用时物化结果；`Count()`、`First()` 等终结操作也会当场求值，但不必遍历所有元素。

### 91. 延迟执行与流式执行是同一回事吗？

**答案：**

不是。延迟执行描述“何时开始”，流式执行描述“是否必须先读取全部输入”。

| 操作 | 执行方式 | 典型行为 |
| --- | --- | --- |
| `Where`、`Select` | 延迟、流式 | 按需读取并产生结果 |
| `OrderBy`、`GroupBy` | 延迟、非流式 | 开始枚举后通常先处理完整输入 |
| `ToList`、`ToArray` | 立即物化 | 保存当前枚举结果 |
| `Any`、`First` | 立即求值 | 得到所需答案后可提前结束 |

因此，对无限序列直接进行完整排序无法得到第一个排序结果。

### 92. `IEnumerable<T>` 与 `IQueryable<T>` 有什么区别？

**答案：**

`Enumerable` 扩展方法通常接收委托，在枚举过程中执行本地逻辑；`Queryable` 方法通常接收表达式树，交由查询提供程序解释或转换，例如转成数据库查询。

`IQueryable<T>` 继承 `IEnumerable<T>`，但不等于所有操作都在数据库执行。提供程序支持哪些表达式、何时请求数据和如何处理不支持的表达式，都取决于实现。不能把任意 C# 方法默认视为可转换的查询。

### 93. `AsEnumerable()` 和 `ToList()` 有什么区别？

**答案：**

`AsEnumerable()` 将后续调用视为面向 `IEnumerable<T>` 的操作，不会立即枚举，也不会自动缓存数据。对查询提供程序而言，它常用于让后续操作采用本地 `Enumerable` 逻辑。

`ToList()` 立即枚举并创建列表，消耗时间和内存，但可以复用这次结果。对远程查询应先尽可能完成可下推的筛选，再决定在哪里切换到本地处理，避免读取不必要的数据。

### 94. `Where()` 和 `Select()` 有什么区别？

**答案：**

`Where` 用谓词筛选元素；`Select` 将每个元素映射成结果，可以改变结果类型，单次投影本身不负责过滤。

```csharp
int[] values = { 1, 2, 3, 4 };
var result = values.Where(x => x > 2).Select(x => $"No.{x}");
// 枚举得到 No.3、No.4
```

不要为了执行副作用而写一个无人枚举的 `Select`；它可能根本不执行。需要逐项操作时，用明确的循环表达意图。

### 95. `Select()` 与 `SelectMany()` 有什么区别？

**答案：**

`Select` 为每个输入生成一个结果，结果可以还是集合；`SelectMany` 将各个输入对应的子序列展开为一个序列。

```csharp
int[][] groups = { new[] { 1, 2 }, new[] { 3 } };
IEnumerable<int[]> nested = groups.Select(g => g);
IEnumerable<int> flat = groups.SelectMany(g => g); // 1、2、3
```

典型场景是从多个订单中取得所有订单项。

### 96. Any、All 和 Count 有什么区别？

**答案：**

`Any()` 表达“存在”，通常找到一个满足条件的元素即可停止；`Count() > 0` 对一般序列可能为了计数遍历更多元素。集合本身提供 `Count` 属性时，直接读取也很合适，不能笼统宣称 `Any()` 对所有来源都更快。

`All(predicate)` 判断是否所有元素满足条件，遇到反例停止。空序列上 `Any()` 为 `false`，`All(predicate)` 为 `true`；若业务要求非空且全部满足，需要同时检查非空，并考虑重复枚举成本。

### 97. `First`、`FirstOrDefault`、`Single`、`SingleOrDefault` 如何选择？

**答案：**

以下按满足条件的元素数量讨论，`OrDefault` 指未显式提供默认值的重载。

| 方法 | 0 个 | 1 个 | 多个 |
| --- | --- | --- | --- |
| `First()` | 抛异常 | 返回该元素 | 返回第一个 |
| `FirstOrDefault()` | 返回 `default(T)` | 返回该元素 | 返回第一个 |
| `Single()` | 抛异常 | 返回该元素 | 抛异常 |
| `SingleOrDefault()` | 返回 `default(T)` | 返回该元素 | 抛异常 |

要求“唯一”用 `Single` 系列，只要求“首个”用 `First` 系列。`default(int)` 是 0，不能用它区分“没有元素”和“第一个元素就是 0”。

### 98. `OrderBy()` 与 `ThenBy()` 有什么区别？

**答案：**

`OrderBy` 建立主排序；`ThenBy` 在已有排序键相等时添加次级排序。再次调用 `OrderBy` 会建立新的主排序，不能替代 `ThenBy`。

```csharp
var sorted = employees
    .OrderBy(e => e.Department)
    .ThenByDescending(e => e.Salary)
    .ThenBy(e => e.Id);
```

LINQ to Objects 的排序稳定；它返回排序后的序列，不会像 `List<T>.Sort()` 那样直接改变原列表顺序。分页时应明确排序及必要的唯一排序键。

### 99. `GroupBy()` 与 `ToLookup()` 有什么区别？

**答案：**

两者都能表示“一个键对应多个元素”。`GroupBy` 返回分组序列，通常延迟执行，开始枚举后读取源数据形成分组；`ToLookup` 立即枚举，创建可按键重复查询的 `ILookup<TKey, TElement>`。

```csharp
var byLength = new[] { "a", "bb", "cc" }.ToLookup(s => s.Length);
// byLength[2] 包含 bb、cc；缺失键返回空序列
```

只需处理一轮分组结果可用 `GroupBy`；需要反复按键取组时可考虑 `ToLookup`。

### 100. `ToDictionary()` 遇到重复键怎么办？

**答案：**

`ToDictionary` 会立即构建字典；键按所选比较器判断重复时抛出异常，不会默认保留最后一个。

应先定义业务规则：键本来就应唯一，则检查或暴露数据错误；允许多值用 `ToLookup`；保留首个可先分组取首个；保留最后一个可在明确循环中通过索引器覆盖。不要未经业务判断就静默丢弃重复项。

### 101. `Distinct()` 如何判断重复？

**答案：**

默认使用 `EqualityComparer<T>.Default`，也可传入 `IEqualityComparer<T>`。普通类如果未定义值相等性，两个字段相同的新对象未必会被视为重复。

```csharp
var names = new[] { "Alice", "ALICE", "Bob" }
    .Distinct(StringComparer.OrdinalIgnoreCase);
```

需要按某个业务键去重时，可投影键再去重，或在目标框架支持时使用 `DistinctBy`。不要把输出顺序当作去重 API 的普遍保证。

### 102. `Concat`、`Union`、`Intersect`、`Except` 有什么区别？

**答案：**

- `Concat`：连接两个序列，保留重复项。
- `Union`：并集，消除重复项。
- `Intersect`：交集，返回双方共有的不同元素。
- `Except`：差集，返回第一序列中存在、第二序列中不存在的不同元素。

例如 `A = [1, 1, 2]`、`B = [2, 3]`：连接包含五项；并集包含 1、2、3；交集包含 2；`A.Except(B)` 包含 1。集合操作仍依赖相等性规则。

### 103. `Cast<T>()` 与 `OfType<T>()` 有什么区别？

**答案：**

`Cast<T>` 尝试逐个转换，不兼容的元素在枚举时导致异常；`OfType<T>` 只保留兼容目标类型的非 null 元素，跳过其余项。

```csharp
object[] data = { 1, "two", null, 3 };
var ints = data.OfType<int>(); // 1、3
// data.Cast<int>().ToList() 遇到 "two" 时失败
```

`Cast<long>()` 不会自动把装箱的 `int` 做数值转换；必要时显式写出 `Select` 转换逻辑。

### 104. Aggregate 如何使用？

**答案：**

`Aggregate` 按指定规则将序列折叠为累计结果。有种子重载从种子开始，空序列可得到种子；无种子重载需要一个初始元素，空序列会抛异常。

```csharp
int[] numbers = { 2, 3, 4 };
int product = numbers.Aggregate(1, (acc, n) => acc * n); // 24
```

求和或平均值优先选择语义直接的 `Sum`、`Average`。空序列行为取决于具体运算和可空重载，不能认为所有聚合操作都返回 0。

### 105. 为什么重复枚举 `IEnumerable<T>` 可能有问题？

**答案：**

`Count()` 后再 `foreach`、`Any()` 后再 `ToList()`，可能多次执行查询。结果可能重复计算、重复读取远程数据、重复产生副作用，或者因为源变化而不一致。

需要复用当前结果时，可一次物化再读取；但要评估数据规模和内存成本，不能为所有序列无条件加 `ToList()`。真正只需单遍处理时，应直接组织一次遍历。

### 106. `Join()`、`GroupJoin()` 和左外连接如何理解？

**答案：**

`Join` 将双方键匹配的元素组合为结果，属于等值内连接；`GroupJoin` 为每个外部元素提供它匹配到的一组内部元素，没有匹配时该组为空。

经典左外连接可通过分组连接，再对组使用 `DefaultIfEmpty()` 展开。这样即使没有匹配也保留外部元素，读取内部结果时必须处理默认值。不要把 `Join` 误认为按索引配对；按位置配对通常使用 `Zip`。

### 107. LINQ 有哪些性能开销？如何优化？

**答案：**

常见开销包括迭代器、闭包与委托、重复枚举、排序／分组缓冲，以及 ToList／ToArray 的结果分配。

优化先减少重复遍历和无用排序，再根据是否需要复用结果决定物化；热点可改成明确循环并复用容器。查询复杂度、数据量和运行时都会影响结果，应以实际测量判断。

### 108. PLINQ 的原理和适用场景是什么？

**答案：**

AsParallel 将查询交给 PLINQ，按数据分区并行执行，再合并结果。适合数据量较大、每项计算较重且彼此独立的 CPU 工作。

分区、调度、同步和保序有成本，小数据可能更慢；委托不要无保护地修改共享状态或调用线程关联 API。异步 I/O 通常使用对应异步接口。

## 八、异步与并发

### 109. `async/await` 有什么作用？底层如何工作？

**答案：**

`async/await` 用于组织异步控制流。异步方法通常被转换为状态机，保存执行进度和需要跨越挂起点的局部状态。

1. 方法通常先在调用线程上同步执行。
2. 遇到尚未完成的 `await` 时，注册后续执行逻辑（continuation），挂起当前异步方法并将控制权交还调用方。
3. 等待的操作完成后，恢复状态机继续执行。
4. 如果等待对象已经完成，可以继续同步执行，不必挂起。

`await` 通常不会阻塞线程直到操作完成；`async` 也不会自动创建后台线程。异步 I/O 可以在没有专用线程持续等待的情况下完成。

### 110. `Task` 和 `Thread` 有什么区别？

**答案：**

- `Thread`：代表一个实际线程，可以显式控制线程生命周期。
- `Task`：代表一个异步操作或未来的结果，通常由线程池调度，但也可以代表不依赖专用线程的异步 I/O。

大多数业务异步任务优先使用 `Task`；需要专门控制线程时才考虑直接使用 `Thread`。

### 111. `Task.Run()` 和直接调用 `async` 方法有什么区别？

**答案：**

```csharp
await DoAsync();               // 直接调用异步方法
await Task.Run(() => Compute()); // 将计算工作提交到线程池
```

直接调用 `DoAsync()` 时，方法通常从当前线程开始执行，直到遇到未完成的 `await` 或返回。

`Task.Run()` 通常会把委托放到线程池执行，适合 CPU 密集型工作；不能简单用它包装所有异步 I/O 来提升性能。

### 112. `SynchronizationContext` 是什么？

**答案：**

它表示特定执行环境的调度抽象，可以通过 `Post()` 等方法将回调交给该环境执行。

在具有 UI 同步上下文的线程上等待 `Task` 时，默认的等待逻辑通常会捕获该上下文，以便后续代码回到对应环境。没有适用同步上下文时，还可能考虑当前任务调度器。

能否恢复到原线程取决于具体上下文和等待器；同步上下文不等于某个固定线程，也不是所有程序都安装了同步上下文。

### 113. 什么是线程安全？如何保证？

**答案：**

多个线程同时访问共享数据时，程序仍然能得到正确结果。

常见方法：

- 使用 `lock` 保护共享状态。
- 使用 `Interlocked` 执行原子操作。
- 使用线程安全集合。
- 尽量采用不可变数据或消息传递，减少共享状态。

### 114. `lock` 的原理是什么？

**答案：**

`lock` 是 C# 的互斥同步机制。对于传统引用类型锁对象的写法，编译器会生成等价的进入和退出监视器逻辑，并确保异常时释放锁。

```csharp
private readonly object _lock = new();

void Add()
{
    lock (_lock)
    {
        count++;
    }
}
```

不要锁定 `this`、字符串或外部可访问对象，以免与其他代码意外竞争同一把锁。

### 115. `lock` 为什么不能用于跨 `await` 的异步临界区？

**答案：**

C# 不允许在 `lock` 语句块内部直接使用 `await`，因为传统 `lock` 依赖同步进入和退出机制，异步挂起可能使临界区跨越不同的执行时刻。

需要异步互斥时，可以使用 `SemaphoreSlim`：

```csharp
private readonly SemaphoreSlim semaphore = new(1, 1);

async Task ExecuteAsync()
{
    await semaphore.WaitAsync();
    try
    {
        await DoAsync();
    }
    finally
    {
        semaphore.Release();
    }
}
```

### 116. 什么是死锁？如何避免？

**答案：**

多个线程互相等待对方持有的资源，导致所有相关线程都无法继续执行。

常见原因是锁顺序不一致、在持锁期间等待其他线程完成等。

解决方法：统一锁获取顺序、缩小锁的范围、避免持锁执行耗时操作，并尽量减少嵌套锁。

### 117. volatile 的作用是什么？与 Interlocked 有什么区别？

**答案：**

C# volatile 约束相关读写的可见性和重排序，用于表达某些跨线程通信。它不使 count++ 这类读—改—写复合操作整体原子。

Interlocked 提供原子递增、交换和比较交换，适合计数器与简单状态更新；多个字段需要维持共同不变量时通常使用 lock。

### 118. 什么是原子操作？为什么 `count++` 不是原子操作？

**答案：**

原子操作对其他线程表现为一个不可分割的操作。

`count++` 通常包含读取、加一、写回多个步骤，多个线程同时执行时可能丢失更新。

```csharp
Interlocked.Increment(ref count);
```

这可以对计数器执行原子递增。

### 119. 什么是 CAS？

**答案：**

CAS（Compare-And-Swap，比较并交换）是原子操作：仅当内存中的值与预期值相等时，才将其更新为新值。

```csharp
int oldValue = Volatile.Read(ref value);

while (true)
{
    int newValue = oldValue + 1;
    int observed = Interlocked.CompareExchange(
        ref value, newValue, oldValue);

    if (observed == oldValue)
        break;

    oldValue = observed;
}
```

CAS 常用于无锁算法，但复杂并发结构还要考虑内存可见性、ABA 问题和对象生命周期。

### 120. 什么是 ABA 问题？

**答案：**

线程读取值 A 后暂停；另一个线程将 A 改成 B，再改回 A。第一个线程恢复时，CAS 发现值仍为 A，就可能误以为期间没有发生变化。

常见解决方法是给值增加版本号，或者采用能够识别状态变化的其他设计。

### 121. ConcurrentDictionary 如何保证线程安全？复合操作怎么处理？

**答案：**

ConcurrentDictionary 提供线程安全的单次字典操作，适合多线程查询和更新。需要原子添加／替换时使用 TryAdd、TryUpdate、GetOrAdd、AddOrUpdate 等接口。

先 ContainsKey 再赋值仍是两个操作，期间状态可能变化。GetOrAdd／AddOrUpdate 的用户工厂可能重复执行，不能把其中副作用当成只发生一次。

### 122. 无锁队列的原理和优缺点是什么？

**答案：**

无锁队列利用原子操作协调多个线程，不依赖传统互斥锁来保证整个队列操作的正确性。

但它不一定更快：CAS 重试、缓存一致性流量和内存分配都可能带来成本。单生产者单消费者（SPSC）队列可以利用更简单的读写索引设计，通常比通用多线程队列更容易优化。

### 123. `Task.WhenAll()` 与 `Task.WhenAny()` 有什么区别？

**答案：**

- `WhenAll`：等待所有指定任务完成；若任务失败，返回的组合任务会记录异常。
- `WhenAny`：等待其中任意一个任务完成，返回最先完成的任务；它不代表其他任务已经结束。

```csharp
await Task.WhenAll(task1, task2);

Task completed = await Task.WhenAny(task1, task2);
await completed; // 继续等待获胜任务，观察其结果、异常或取消状态
```

`WhenAny` 不会自动取消剩余任务，需要通过取消令牌或其他机制处理。

### 124. CancellationToken 如何取消异步任务？

**答案：**

CancellationToken 是协作式取消信号。调用方发出取消请求，执行方通过传递 token、检查 IsCancellationRequested 或调用 ThrowIfCancellationRequested 响应。

它不会强制终止任意代码。任务应在合适节点检查并清理资源；共享底层操作还需区分某个等待者取消和整个任务取消。

### 125. 为什么通常不推荐使用 `async void`？

**答案：**

`async void` 无法像 `Task` 一样被调用方等待，也无法通过返回值统一观察其完成状态和异常。

一般业务异步方法应返回 `Task` 或 `Task<T>`。`async void` 主要适用于必须匹配 `void` 签名的事件处理器，并且需要明确处理异常。

### 126. ConfigureAwait(false) 有什么作用？

**答案：**

在相应 await 中，不要求后续代码回到捕获的同步上下文。库代码常用它减少对调用环境的依赖。

它不强制新建线程或切到线程池，已完成的等待也可能同步继续。后续需要访问 UI／Unity 主线程对象时，必须按环境恢复到正确上下文。

### 127. 为什么 `.Result`、`.Wait()` 可能造成死锁？

**答案：**

如果 UI 线程同步等待异步任务，而该任务挂起后的代码又需要回到这个被阻塞的 UI 上下文，两者就可能互相等待。

优先让调用链保持异步，使用 `await`。没有同步上下文也不代表同步等待没有成本：大量阻塞可能造成线程池饥饿。不要把局部加上 `ConfigureAwait(false)` 当作修复所有同步等待问题的通用方法。

### 128. `Task<T>` 与 `ValueTask<T>` 如何选择？

**答案：**

`Task<T>` 是引用类型的异步结果表示，易于保存、共享和组合。`ValueTask<T>` 是值类型包装，可表示立即完成的结果，也可包装任务或可复用的异步源。

高频且经常同步完成的路径可能受益，但它增加使用约束和复制成本，不能认为一定更快。通常只消费一次 `ValueTask<T>`；若要反复等待或参与基于 `Task` 的组合，可调用一次 `AsTask()` 并保存所得任务。默认优先 `Task<T>`，有测量依据再选择。

### 129. `Thread.Sleep()` 与 `Task.Delay()` 有什么区别？

**答案：**

`Thread.Sleep` 阻塞当前线程；`Task.Delay` 返回在延迟结束后完成的任务，配合 `await` 可以挂起方法而不占用一个持续等待的线程。

```csharp
await Task.Delay(500, cancellationToken);
```

两者都不是精密计时器。不要用 `.Wait()` 同步阻塞 `Task.Delay` 来假装异步，也不要用固定延时替代正确的线程同步。

## 九、异常处理

### 130. `throw;` 与 `throw ex;` 有什么区别？

**答案：**

在 `catch` 中重新抛出当前异常时，`throw;` 保留原始调用栈，`throw ex;` 会重置栈信息中的重新抛出位置，损害故障定位。

```csharp
try
{
    Execute();
}
catch (Exception ex)
{
    Log(ex);
    throw;
}
```

需要跨位置保存并重新抛出异常时，可考虑 `ExceptionDispatchInfo`。

### 131. finally 在什么时候执行？

**答案：**

正常离开 `try`、执行 `return` 或异常展开时，都会执行相关 `finally`，因此适合资源清理。

但“任何情况下都执行”过于绝对：进程被强制终止、`Environment.FailFast` 等情况下不能依赖它。`finally` 中再次抛异常还可能掩盖原始异常，所以清理逻辑也要谨慎设计。

### 132. 什么是异常过滤器？什么时候使用 `TryParse` 而不是异常？

**答案：**

`catch (...) when (...)` 只有在过滤条件成立时才处理该异常，便于根据错误信息决定是否接管。过滤器不应承担关键状态修改等副作用。

```csharp
catch (IOException ex) when (ShouldRetry(ex))
{
    // 只处理满足条件的 I/O 错误
}
```

可预期的输入失败优先使用 `TryParse`、`TryGetValue` 等返回状态的 API；不要用频繁抛异常表达普通业务分支，也不要空 `catch` 吞掉无法处理的错误。

## 十、设计与测试基础

### 133. 什么是里氏替换原则？

**答案：**

子类对象应能替代父类对象使用，并保持父类对调用者承诺的行为。

子类不应加强前置条件、削弱后置条件或破坏不变量。例如父类承诺可写，子类却对所有写入都抛异常，就不满足替换原则。它强调行为契约，不只是语法上能赋值给父类引用。

### 134. Mock 与 Stub 有什么区别？

**答案：**

Stub 主要提供预设返回值或响应，用于控制被测对象的依赖环境；Mock 主要验证交互预期，例如是否以某参数调用某方法。

这不是“小测试／大测试”的区别，同一工具也可能同时提供两种能力。优先验证可观察结果；只有交互本身属于业务契约时才断言调用次数，避免测试与实现细节过度绑定。

## 十一、运行时与执行模型

### 135. C#、IL、CLR、JIT 与 AOT 分别是什么？

**答案：**

C# 是语言，IL／CIL 是常见中间表示，CLR 是提供类型、执行、GC、异常与互操作等服务的运行时。常见执行链路为源码转换为包含 IL 与元数据的程序集，再由 JIT 在运行时生成目标机器代码。

AOT 则把目标代码生成提前到运行之前，具体部署也可能混合不同策略。因此“C# 必须依靠 JIT 才能运行”不准确；Unity 的 Mono／IL2CPP 等后端应按项目平台分别理解。

JIT 的典型行为是按需编译：方法第一次被调用时才把 IL 翻译成当前平台的机器码并缓存到内存，后续调用直接执行缓存代码，不再重复编译；没有调用到的方法永远不编译。好处是一份 IL 可在 x64、ARM 等不同架构上运行，现代 .NET 的分层编译还会先快速生成弱优化代码保证启动速度，热点方法再由后台重新做内联、去虚拟化、循环展开等更强优化，并能利用当前 CPU 指令集（如 AVX）。

代价方面：首次调用包含编译开销，游戏主线程第一次走到复杂方法可能出现 JIT 卡顿；运行时需保留 IL 元数据，占用内存；iOS 与游戏主机禁止进程在运行时动态生成机器码，这些平台只能用 AOT。Unity 项目中缓解 JIT 卡顿的常用手段：在场景加载阶段主动预热热点方法、改用 IL2CPP 消除 JIT、减少高频路径上的泛型实例化以降低需编译的代码量。

托管 GC 自动回收不可达对象内存，不代表文件、连接、原生句柄等都能及时释放；仍需 `IDisposable`／`using` 等明确管理资源。与 C++ 的对比见第 22 题和 [C++ 专题](C++高频题整理.md)。

Mono JIT 与 IL2CPP 的执行流程对比见 [Unity 专题](Unity高频题整理.md)第 79 题。

### 136. IL2CPP 的工作原理是什么？

**答案：**

IL2CPP 是 Unity 的 AOT 编译后端，整条链路发生在打包阶段：C# 先由 Roslyn 编译成 IL，IL2CPP 转换器把 IL 翻译成等价的 C++ 源码，再交给目标平台的本地工具链（clang／MSVC 等）编译成原生机器码；运行时由 il2cpp 运行时库（libil2cpp）提供 GC、元数据、异常、线程等托管服务。运行时不再有 JIT 编译环节。

与 Mono JIT 相比的特点：

- 运行前已完成代码生成，没有首次调用的 JIT 卡顿，运行时性能更稳定；代价是打包时间明显变长，原生调试更复杂。
- 苹果系统与游戏主机禁止运行时生成机器码，这些平台必须走 IL2CPP。
- 静态转换配合托管代码裁剪（Stripping）会移除静态分析认为未使用的类型与成员，反射、`JsonUtility` 之外按字符串间接构造的序列化路径可能失效，需用 `link.xml` 或 `[Preserve]` 保留。
- 不支持 `Reflection.Emit` 等运行时动态代码生成；泛型在值类型等场景需要独立机器码，静态分析覆盖不到的泛型实例化可能出现 AOT 泛型缺失报错，需要显式引用或生成泛型提取代码。
- 它仍是托管运行时，并非“转成 C++ 就没有 GC 和托管语义”。

追问题“怎么解决 JIT 卡顿”的答案与本题直接相关：预热只能把编译开销挪到加载阶段，IL2CPP 从机制上消除 JIT，是 iOS 等平台唯一选择。

参考：[Unity：IL2CPP 简介](https://docs.unity3d.com/6000.0/Documentation/Manual/il2cpp-introduction.html)。

## 参考资料

- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)
- [Microsoft：大对象堆](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/standard/linq/deferred-execution-lazy-evaluation)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.iqueryable-1)
- [AsEnumerable 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.asenumerable)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/projection-operations)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.singleordefault)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.orderby)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.tolookup)
- [ToDictionary 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.todictionary)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/set-operations)
- [Cast 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.cast)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1851)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/join-operations)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask)
- [Microsoft 官方说明](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft：托管执行过程](https://learn.microsoft.com/en-us/dotnet/standard/managed-execution-process)
