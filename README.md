# ⚡ EasyECS

**OOP-compatible SoA data layout optimizer for Unity**

EasyECS 是一个面向 Unity 的 **OOP 兼容 SoA（Structure of Arrays）数据布局优化插件**。

它不是一套新的 ECS 游戏框架，也不是 Unity Entities / DOTS 的替代品。EasyECS 不要求项目改造成 Entity / Component / System 架构，也不引入 Archetype、Query、Scheduler、Job System 等完整 ECS 概念。

它只解决一个问题：

> **尽量保持现有 OOP 业务代码的写法，只改变热点数据的物理存储布局，让需要连续访问的字段进入更适合 CPU Cache 的 SoA 存储。**

通过 `[ECS]` / `[NotECS]` 标记普通 `struct`，EasyECS 会使用 **C# Source Generator** 在编译期生成：

```text
SoA / AoS Hybrid Storage
<Type>Ref
<Type>ECSList
<Type>ECSDictionary<TKey>
Direct Column
Editor Lifecycle / Bounds / Ref Safety Check
```

业务代码仍然可以写成：

```csharp
RoleDataRef role = roles[i];
role.mHP -= damage;
role.mPositionX += role.mSpeed;
```

而底层存储已经可以变成：

```text
mHP[]
mSpeed[]
mPositionX[]
mPositionY[]

mAoS[]
 ├─ mID
 ├─ mModelID
 └─ mCamp
```

---

## 📌 项目定位

EasyECS 适合这些场景：

- 已经存在大量 OOP 业务代码，不希望整体迁移到 DOTS / Entities。
- 有大量角色、怪物、子弹、Buff、技能状态、AI 状态等结构化数据。
- Profiler 已经确认某些循环存在明显的数据访问热点。
- 希望获得 SoA 数据布局的优势，但不希望引入完整 ECS 架构。
- 希望同一套业务 API 在 Unsafe / SafeSpan / 兼容后端之间自动切换。

EasyECS **不适合**这些需求：

- 需要 Archetype / Chunk / Query 系统。
- 需要 Job System / Burst 调度框架。
- 需要完整 Entity 生命周期框架。
- 希望替代 Unity Entities / DOTS。

EasyECS 的定位始终是：

```text
OOP Business Code
        ↓
Source Generator
        ↓
Generated Data Layout / Containers
        ↓
SoA / AoS Hybrid Storage
```

---

## ⚠️ 关于本仓库

`ZHOURUIH/EasyECS` 当前作为 EasyECS 的独立展示与文档入口。

EasyECS 的实际源码维护在：

```text
https://github.com/ZHOURUIH/MyFramework
```

源码目录：

```text
Packages/com.zhourui.easyecs
```

因此安装时请使用 **MyFramework 子目录 Git URL**，不要直接把当前展示仓库作为 UPM Package 安装。

---

## 📦 安装

Unity 中打开：

```text
Window
    Package Manager
        +
            Install package from git URL...
```

### GitHub

```text
https://github.com/ZHOURUIH/MyFramework.git?path=/Packages/com.zhourui.easyecs
```

### Gitee

```text
https://gitee.com/inothingtodo/MyFramework.git?path=/Packages/com.zhourui.easyecs
```

Package Name：

```text
com.zhourui.easyecs
```

---

## 🚀 快速开始

### 1. 定义普通 Struct

```csharp
using EasyECS;

[ECS]
public struct RoleData
{
	public int mHP;
	public float mSpeed;
	public float mPositionX;
	public float mPositionY;

	[NotECS] public int mID;
	[NotECS] public int mModelID;
	[NotECS] public int mCamp;
}
```

EasyECS 会自动生成对应代码，例如：

```text
RoleDataAoSBlock
RoleDataStorage
RoleDataRef
RoleDataECSList
RoleDataECSDictionary<TKey>

getHPColumn()
getSpeedColumn()
getPositionXColumn()
getPositionYColumn()
...
```

### 2. ECSList

```csharp
RoleDataECSList roles = new RoleDataECSList(1024);

roles.Add(new RoleData
{
	mHP = 100,
	mSpeed = 5.0f,
	mPositionX = 0.0f,
	mPositionY = 0.0f,
	mID = 1001,
	mModelID = 10,
	mCamp = 1,
});

roles[0].mHP -= 10;

RoleDataRef role = roles[0];
role.mPositionX += role.mSpeed;
```

### 3. ECSDictionary

```csharp
RoleDataECSDictionary<int> roles = new RoleDataECSDictionary<int>(1024);

roles.Add(1001, new RoleData
{
	mHP = 100,
	mSpeed = 5.0f,
	mID = 1001,
});

roles[1001].mHP -= 10;

if (roles.TryGetValue(1001, out RoleDataRef role))
{
	role.mPositionX += role.mSpeed;
}
```

---

## 🧩 `[ECS]` / `[NotECS]` 规则

### Struct 标记 `[ECS]`

字段默认进入 ECS / SoA：

```csharp
[ECS]
public struct RoleData
{
	public int mHP;
	public float mSpeed;

	[NotECS] public int mID;
}
```

近似布局：

```text
mHP[]
mSpeed[]

mAoS[]
 └─ mID
```

### Struct 标记 `[NotECS]`

字段默认进入 AoS，只有显式 `[ECS]` 字段进入 SoA：

```csharp
[NotECS]
public struct RoleData
{
	[ECS] public int mHP;
	[ECS] public float mSpeed;

	public int mID;
	public int mCamp;
}
```

### 字段标记优先于 Struct 默认规则

规则可以总结为：

```text
Struct [ECS]
→ Field 默认 ECS
→ Field [NotECS] 覆盖为 AoS

Struct [NotECS]
→ Field 默认 AoS
→ Field [ECS] 覆盖为 SoA
```

同一个声明同时出现：

```csharp
[ECS]
[NotECS]
```

会产生 Source Generator 错误。

---

## 🧠 Hybrid Storage

EasyECS 不要求一个 `struct` 的所有字段都必须是 unmanaged。

例如：

```csharp
[ECS]
public struct RoleRuntimeData
{
	public int mHP;
	public float mSpeed;
	public string mName;
	public object mPayload;

	[NotECS] public int mID;
}
```

当允许 Unsafe，并且结构体中存在适合 Native Storage 的字段时，EasyECS 可以生成 Hybrid Storage：

```text
Unmanaged ECS Fields
→ Native SoA

Managed ECS Fields
→ Managed SoA Array

AoS Fields
→ 如果全部 unmanaged，可进入 Native AoS
→ 如果包含 managed 字段，则整个 AoS Block 使用 Managed Array
```

因此：

> **一个 `string` 或 `object` 字段不会强迫整个 Struct 放弃 Unsafe Backend。**

只有不能进入 Native Memory 的字段自身使用托管存储。

---

## ⚙ Backend

EasyECS 在 Source Generator 阶段自动选择 Backend。

逻辑近似为：

```text
ECS_FORCE_SAFE_REGISTRY
        ↓
SafeRegistry

否则 Allow Unsafe Code = true
并且存在 Native Storage
        ↓
Unsafe

否则当前编译环境支持 Span<T>
        ↓
SafeSpan

否则
        ↓
SafeRegistry
```

### Unsafe

用于存在 unmanaged 热点字段并允许 Unsafe 的情况。

特点：

- Native SoA。
- Native AoS（满足 unmanaged 条件时）。
- Pointer-backed Direct Column。
- 支持 managed 字段与 native 字段混合的 Hybrid Storage。
- Resize 后已有 Ref 仍可保持指向稳定 Storage。

### SafeSpan

不允许 Unsafe 或没有 Native Storage 时的高性能安全路径。

特点：

- Managed Array Storage。
- Span / ReadOnlySpan 友好的访问路径。
- 不使用 Native Pointer。
- 保持与 Unsafe 基本一致的业务 API。

### SafeRegistry

兼容 Backend。

可以通过 Scripting Define Symbols 强制启用：

```text
ECS_FORCE_SAFE_REGISTRY
```

适合：

- 调试兼容路径。
- 不具备 Span 支持的编译环境。
- 需要显式验证最保守 Backend 的情况。

---

## ⚡ 三种访问层级

EasyECS 不要求所有代码都使用 Direct Column。

推荐根据热点程度选择访问方式。

### 1. 简单单字段

```csharp
roles[i].mHP -= 1;
```

适合普通逻辑。

### 2. 多字段访问

推荐缓存 Ref：

```csharp
RoleDataRef role = roles[i];

float speed = role.mSpeed;
role.mHP -= 1;
role.mPositionX += speed;
role.mPositionY -= speed;
```

避免同一轮业务逻辑反复构造访问路径。

### 3. 极端热点循环

使用 Direct Column：

```csharp
var hp = roles.getHPColumn();
var speed = roles.getSpeedColumn();
var positionX = roles.getPositionXColumn();
var positionY = roles.getPositionYColumn();

for (int i = 0; i < roles.Count; ++i)
{
	hp[i] -= 1;
	float curSpeed = speed[i];
	positionX[i] += curSpeed;
	positionY[i] -= curSpeed;
}
```

推荐层级：

```text
普通业务逻辑
→ Ref

简单单字段
→ list[i] / dictionary[key]

Profiler 确认的极端热点循环
→ Direct Column
```

---

## 📚 ECSList

生成类型：

```csharp
<Type>ECSList
```

主要接口：

```text
Count
Capacity
Add
Insert
RemoveAt
RemoveAtSwapBack
Clear
Indexer
Direct Column
Dispose
```

### Add

```csharp
roles.Add(value);
```

摊销 O(1)。

### Insert

```csharp
roles.Insert(index, value);
```

语义与 `List<T>.Insert` 一致：

- `index == Count` 合法。
- 保持元素顺序。
- O(n)。

### RemoveAt

```csharp
roles.RemoveAt(index);
```

语义与 `List<T>.RemoveAt` 一致：

- 保持元素顺序。
- O(n)。

### RemoveAtSwapBack

```csharp
roles.RemoveAtSwapBack(index);
```

不保持顺序：

```text
删除 index
↓
最后一个元素移动到 index
↓
Count--
```

适合不关心顺序的热点容器，复杂度 O(1)。

---

## 🗂 ECSDictionary

生成类型：

```csharp
<Type>ECSDictionary<TKey>
```

内部结构：

```text
Dictionary<TKey, int>
        ↓
    dense index
        ↓
<Type>ECSList
```

Key 查询仍由 BCL `Dictionary<TKey,int>` 负责，Value 使用 EasyECS 连续存储。

主要接口：

```text
Add
TryAdd
ContainsKey
Indexer
TryGetValue
TryGetIndex
Remove
Clear
Count
Capacity
Comparer
getKeyAt
getValueAt
Keys
Values
foreach
Direct Column
Dispose
```

### 随机访问

```csharp
roles[1001].mHP -= 10;
```

或者：

```csharp
if (roles.TryGetValue(1001, out RoleDataRef role))
{
	role.mHP -= 10;
}
```

### Dense 遍历

```csharp
for (int i = 0; i < roles.Count; ++i)
{
	RoleDataRef role = roles.getValueAt(i);
	role.mHP -= 1;
}
```

### Keys

```csharp
foreach (int key in roles.Keys)
{
	// ...
}
```

Player 下对支持 Span 的环境使用高性能只读 Key 遍历路径。

### Values

```csharp
foreach (RoleDataRef role in roles.Values)
{
	role.mHP -= 1;
}
```

Values 返回可直接修改底层数据的 Ref。

### Key + Value

```csharp
foreach (var item in roles)
{
	int key = item.Key;
	RoleDataRef value = item.Value;
}
```

### Remove

Dictionary 的 `Remove` 使用 dense swap-back：

```text
Key -> dense index
删除目标
最后一个 Value / Key 移动到空位
更新移动 Key 的 dense index
```

因此：

> **ECSDictionary 不保证遍历顺序稳定。**

如果业务依赖顺序，应单独维护顺序数据。

---

## 🔗 Ref 生命周期

EasyECS 的 Ref 是：

> **位置引用（position reference），不是永久实体身份句柄。**

### Resize

已有 Ref 可以跨 Resize 保持有效。

### Insert

```text
Insert(index)

旧 Ref index < 插入位置
→ 保持有效

旧 Ref index >= 插入位置
→ 位置已经改变，不应继续当成原实体引用

Insert(Count)
→ 现有元素位置不变
```

### RemoveAt

```text
RemoveAt(index)

旧 Ref index < 删除位置
→ 保持有效

旧 Ref index >= 删除位置
→ 位置可能发生变化
```

### RemoveAtSwapBack / Dictionary Remove

被删除位置和被移动元素相关的旧位置引用都不应继续作为原实体身份使用。

### Clear / Dispose

所有旧 Ref 失效。

Editor 生成代码会增加生命周期、版本和边界检查，尽量让无效 Ref 尽早暴露。

---

## 📊 Direct Column 生命周期

Direct Column 是临时字段视图，不是长期持有的 Collection。

例如：

```csharp
var hp = roles.getHPColumn();
```

拿到 Column 后，如果发生结构变化：

```text
Add
Insert
RemoveAt
RemoveAtSwapBack
Clear
Resize
Dispose
```

都不应该继续使用旧 Column。

正确方式：

```csharp
var hp = roles.getHPColumn();

for (int i = 0; i < roles.Count; ++i)
{
	hp[i] -= 1;
}

// 结构变化后重新获取
roles.Add(value);
hp = roles.getHPColumn();
```

---

## 🛡 Editor Safety

Editor 下生成代码会保留更多检查，例如：

- Dispose 后访问检查。
- Index 越界检查。
- Ref 生命周期检查。
- Column 结构版本检查。
- Enumerator 结构版本检查。
- Native allocation leak tracking。
- Dictionary / List 结构变化检测。

Player 下不会承担这些 Editor-only 检查成本。

---

## 🧹 Dispose

三个 Backend 都提供统一：

```csharp
Dispose();
```

推荐：

```csharp
RoleDataECSList roles = new RoleDataECSList();

try
{
	// ...
}
finally
{
	roles.Dispose();
}
```

Unsafe Backend 会释放 Native Memory。

SafeSpan / SafeRegistry 也保留统一 Dispose 与生命周期失效语义。

重复 Dispose 已纳入 Runtime Unit Test。

---

## 🧵 线程安全

EasyECS 容器不是线程安全容器。

不要并发执行结构修改：

```text
Add
Insert
RemoveAt
RemoveAtSwapBack
Remove
Clear
Resize
Dispose
```

也不要在一个线程进行结构变化时，让其他线程继续持有旧 Ref / Column 访问同一容器。

如果项目需要多线程访问，应由业务层自行保证同步与生命周期。

---

## 🧪 Source Generator Diagnostics

当前主要诊断：

| Code | 说明 |
|---|---|
| `ECS001` | `[ECS]` / `[NotECS]` 标签冲突 |
| `ECS002` | 不支持生成的 ECS 类型 |
| `ECS003` | 不支持生成的字段 |
| `ECS004` | 生成的 Direct Column 方法名称冲突 |

Generator 会在编译阶段直接报告问题，而不是把错误拖到运行时。

---

## 🧪 Benchmark Sample

安装 Package 后执行：

```text
EasyECS
    Import Benchmark Sample
```

会导入 Benchmark Sample。

当前 Sample 包括：

```text
RoleDataBenchmark
RoleDataDictionaryBenchmark
RoleDataDictionaryEnumeratorBenchmark
RoleDataListStructuralBenchmark
EasyECSRuntimeUnitTest
```

主要用于：

- List 正确性与性能测试。
- Dictionary 正确性与性能测试。
- Ref / Direct Column 测试。
- Keys / Values / Enumerator 测试。
- Insert / RemoveAt / SwapBack 测试。
- Managed / Hybrid Storage 测试。
- Backend 选择测试。
- Runtime 生命周期回归。

---

## ✅ 1.1.0 最终验收

最终 Runtime Unit Test：

```text
Unsafe   : 59 / 59 PASS
SafeSpan : 59 / 59 PASS
```

最终性能验收环境：

```text
Unity        : 6000.3.21f1
Platform     : Windows x64 Player
Scripting    : IL2CPP
Build        : Release
Graphics     : Direct3D 12
GPU          : NVIDIA GeForce RTX 2060
VRAM         : 5955 MB
CPU Threads  : 32

EntityCount      : 500000
SampleCount      : 15
WarmupCount      : 3
RandomWriteCount : 50000
```

> Benchmark 是微基准测试，用于比较同一环境、同一业务循环下的访问路径，不代表所有项目中的绝对性能。

---

## 📈 代表性性能结果

以下数据为最终封版日志的 Median。

### ECSList：修改 1 个字段

单位：`ns / entity`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `RoleData[]` | 0.588 | 0.570 |
| `ECS list[i]` | 0.486 | 0.713 |
| `ECS Ref` | 0.488 | 0.713 |
| `ECS Direct` | **0.320** | **0.366** |
| `List<RoleData>` | 7.946 | 7.610 |

### ECSList：访问 2 个字段

单位：`ns / entity`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `RoleData[]` | 0.559 | 0.556 |
| `ECS list[i]` | 0.708 | 1.237 |
| `ECS Ref` | 0.534 | 0.888 |
| `ECS Direct` | **0.236** | **0.440** |

### ECSList：访问 4 个字段

单位：`ns / entity`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `RoleData[]` | 1.087 | 0.917 |
| `ECS list[i]` | 1.718 | 2.832 |
| `ECS Ref` | 1.150 | 1.762 |
| `ECS Direct` | **0.617** | **0.969** |

### 4 字段读写访问路径拆解

单位：`ns / entity`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| Raw SoA arrays | 0.946 | 0.899 |
| repeated `list[i]` | 1.720 | 2.833 |
| Local Ref | 1.150 | 1.764 |
| Ref + cache repeated input | 0.997 | 1.525 |
| Direct Column | **0.617** | **0.969** |

这组数据对应 EasyECS 推荐的访问层级：

```text
单字段
→ list[i]

多字段
→ Local Ref

重复读取同一个输入字段
→ Local Ref + 局部变量缓存

极端热点
→ Direct Column
```

---

## 📈 ECSDictionary 性能

### 随机 Key 读取

单位：`ns / op`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 19.380 | **15.760** |
| ECS Inline Indexer | 16.062 | 16.425 |
| ECS Local Ref | **15.858** | 16.166 |
| ECS TryGetValue | 15.920 | 16.709 |

随机 Key 访问仍然受 Hash Lookup 主导，因此 EasyECS 不承诺每一种随机查询都显著快于 BCL Dictionary。

EasyECS 的主要优势来自：

> **Key Lookup + Dense Value Storage + 后续连续批处理。**

### 随机 Key 修改

单位：`ns / op`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 24.754 | 24.722 |
| ECS Inline Indexer | 15.404 | 15.772 |
| ECS Local Ref | 15.826 | 15.708 |
| ECS TryGetValue | **14.980** | **15.254** |

### Dense 全量更新 + 10% 随机 Key 修改

| 路径 | Unsafe Median | SafeSpan Median |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 13.003 ms | 12.405 ms |
| Manual `IndexMap + RoleData[]` | 1.348 ms | 1.250 ms |
| `ECS Direct + LocalRef` | **0.893 ms** | **1.021 ms** |

对应比例：

```text
Unsafe:
Standard Dictionary / ECS = 14.56x
Manual / ECS              = 1.51x

SafeSpan:
Standard Dictionary / ECS = 12.15x
Manual / ECS              = 1.22x
```

这类场景正是 EasyECS 的主要目标：

```text
需要随机 Key 定位
+
每帧又需要对大量 Value 连续更新
```

---

## 📈 Dictionary 遍历

### Keys

单位：`ns / op`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `for + getKeyAt` | 0.352 | 0.353 |
| `foreach dict.Keys` | 0.377 | 0.361 |

### Values 修改 1 个字段

单位：`ns / op`

| 路径 | Unsafe | SafeSpan |
|---|---:|---:|
| `for + getValueAt` | 0.723 | 0.897 |
| `foreach dict + item.Value` | 0.411 | 0.638 |
| `foreach dict.Values` | 0.395 | 0.549 |
| Direct Column | **0.364** | **0.486** |

最终版本中，`dict.Values` 已经接近 Direct Column，不再存在早期自定义 Enumerator 的高额开销。

---

## 📈 结构操作

最终结构性能门槛：

```text
存在真实数据移动的 Insert / RemoveAt
ECS / List <= 1.05
```

最终 Unsafe：

```text
Insert Head        ECS/List = 0.989x PASS
Insert Middle      ECS/List = 0.982x PASS
RemoveAt Head      ECS/List = 0.312x PASS
RemoveAt Middle    ECS/List = 0.315x PASS

Hybrid Insert      ECS/List = 0.922x PASS
Hybrid RemoveAt    ECS/List = 0.842x PASS
```

最终 SafeSpan：

```text
Insert Head        ECS/List = 1.014x PASS
Insert Middle      ECS/List = 1.018x PASS
RemoveAt Head      ECS/List = 0.992x PASS
RemoveAt Middle    ECS/List = 0.993x PASS

Hybrid Insert      ECS/List = 0.927x PASS
Hybrid RemoveAt    ECS/List = 0.899x PASS
```

Tail Add/Insert/Remove 属于数纳秒级操作，因此不使用百分比作为硬性性能门槛。

---

## 📦 Capacity / Resize

EasyECS 的 SoA 结构在 Resize 时可能需要扩展多个字段列。

对于预计会达到较大数量的容器，推荐：

```csharp
RoleDataECSList roles = new RoleDataECSList(expectedCount);
```

或者：

```csharp
RoleDataECSDictionary<int> roles = new RoleDataECSDictionary<int>(expectedCount);
```

也就是：

> **已知规模时优先预留 Capacity，而不是依赖多次自动扩容。**

Resize 是低频结构操作，不建议为了极少发生的 Resize 牺牲日常访问路径的简单性与连续列布局。

---

## 🧹 关于 GC

EasyECS 的热点 API 设计目标之一是避免不必要的托管分配。

但当前封版 Release Player 中，`ProfilerRecorder("GC.Alloc")` 的正向 SelfCheck 无法工作，因此 **0.1.0 README 不把无效的 0 结果写成“实测 0 GC”结论**。

如果需要精确检查项目中的 managed allocation，建议：

```text
Development Build
+
Unity Profiler
+
CPU Usage / GC.Alloc
```

README 中的性能表只发布已经有效验证的 CPU 时间数据。

---

## 🏗 Source Generator

源码目录：

```text
Packages/com.zhourui.easyecs/SourceGenerator~
```

发布 Package 中 Analyzer 位于：

```text
Packages/com.zhourui.easyecs/Analyzers/ECSGenerator.dll
```

普通使用者不需要手动编译 Generator。

维护 Generator 时再进入 `SourceGenerator~` 编译并更新 Analyzer DLL。

Source Generator 独立使用：

```text
netstandard2.0
Microsoft.CodeAnalysis.CSharp 4.3.0
```

---

## 📁 Package 目录

```text
Packages/com.zhourui.easyecs
├── Analyzers
│   └── ECSGenerator.dll
├── Editor
│   ├── EasyECS.Editor.asmdef
│   └── EasyECSMenu.cs
├── Runtime
│   ├── EasyECS.Runtime.asmdef
│   └── ECSAttribute.cs
├── Samples~
│   └── Benchmark
├── SourceGenerator~
│   ├── ECSGenerator
│   └── ECSGeneratorTest
├── CHANGELOG.md
├── LICENSE.md
├── README.md
└── package.json
```

---

## 🔄 HybridCLR / IL2CPP

EasyECS 的工作阶段：

```text
业务代码
    ↓
C# Source Generator
    ↓
生成普通 C# 代码
    ↓
Unity C# 编译
    ↓
HybridCLR / Obfuz / IL2CPP
```

EasyECS 不依赖 IL Post Processor 修改已编译 IL。

生成结果最终仍然是普通 C# 类型，因此可以继续进入 Unity 后续编译流程。

---

## ⚠️ 已知限制

0.1.0 当前限制：

- 只针对 `struct` 数据布局优化。
- 不提供完整 ECS Entity / Component / System 框架。
- 容器不是线程安全的。
- Ref 是位置引用，不是永久实体身份。
- Direct Column 不应跨结构变化长期持有。
- `ECSDictionary.Remove` 使用 swap-back，不保证遍历顺序。
- 大容器应尽量提前设置 Capacity。
- SafeRegistry 是兼容后端，不是主要性能目标。
- GC 的最终精确数据应使用 Development Build + Unity Profiler 检查。
- Source Generator 对不支持的声明会直接产生编译诊断。

---

## 🗺 后续方向

1.1.0 发布后，EasyECS 核心性能路径冻结。

后续优先级应是：

```text
稳定性
文档
真实项目验证
兼容性
API完整度
```

而不是继续为了极小的纳秒差异增加 Generator 复杂度。

---

## 🤝 反馈

Issue：

```text
https://github.com/ZHOURUIH/EasyECS/issues
```

源码仓库：

```text
https://github.com/ZHOURUIH/MyFramework
```

EasyECS Package：

```text
Packages/com.zhourui.easyecs
```

---

## 📄 License

MIT License

Copyright (c) 2026 zhourui
