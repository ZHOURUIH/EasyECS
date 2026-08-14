# ⚡ EasyECS

一个面向 Unity 的 **OOP 兼容 SoA（Structure of Arrays）数据布局优化插件**。

EasyECS 并不是一套完整的 ECS Framework，也不是 Unity Entities / DOTS 的替代品。

它的目标是：

> **尽量保持原有 OOP 项目的代码结构和开发习惯，只改变热点数据的底层存储布局，以较低的改造成本获得 SoA 更好的 CPU Cache 局部性和连续内存访问性能。**

对于需要优化的数据，只需要添加 `[ECS]` / `[NotECS]`，EasyECS 会通过 **C# Source Generator** 自动生成 SoA Storage、Ref、ECSList、Direct Column 等代码。

---

## ⚠️ 关于本仓库

**当前仓库不是 EasyECS 的源码维护仓库。**

本仓库主要用于：

* EasyECS 项目展示
* README 与使用说明
* 独立项目入口

EasyECS 的实际源码目前维护在：

```text
https://github.com/ZHOURUIH/MyFramework
```

具体目录：

```text
Packages/com.zhourui.easyecs
```

因此：

> **安装 EasyECS 时请使用下面提供的 MyFramework 子目录 Git URL，而不是直接安装当前仓库。**

---

# 📦 安装

打开 Unity：

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

GitHub 访问不稳定时可以使用：

```text
https://gitee.com/inothingtodo/MyFramework.git?path=/Packages/com.zhourui.easyecs
```

Package Name：

```text
com.zhourui.easyecs
```

---

# 🚀 快速开始

定义一个普通 Struct：

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

EasyECS 会在编译阶段自动生成对应的：

```text
RoleDataStorage
RoleDataRef
RoleDataECSList
Direct Column
...
```

之后即可使用：

```csharp
using (RoleDataECSList list = new RoleDataECSList(1024))
{
	list.Add(new RoleData
	{
		mHP = 100,
		mSpeed = 5.0f,
		mID = 1,
	});

	RoleDataRef role = list[0];
	role.mHP -= 10;
	role.mPositionX += role.mSpeed;
}
```

业务逻辑仍然是普通 OOP 风格：

```csharp
RoleDataRef role = list[i];
role.mPositionX += role.mSpeed;
role.mHP -= damage;
```

而底层数据已经按照 SoA 方式存储。

---

# ⚡ 三种访问方式

EasyECS 可以根据代码热点程度选择不同的访问方式。

### 单字段

```csharp
list[i].mHP -= 1;
```

### 多字段

推荐缓存 Ref：

```csharp
RoleDataRef role = list[i];
role.mPositionX += role.mSpeed;
role.mPositionY += role.mSpeed;
role.mHP -= 1;
```

### 极端热点批处理

直接访问 Column：

```csharp
var hp = list.getHPColumn();
var speed = list.getSpeedColumn();
var positionX = list.getPositionXColumn();

for (int i = 0; i < list.Count; ++i)
{
	hp[i] -= 1;
	positionX[i] += speed[i];
}
```

推荐：

```text
普通业务逻辑
    ↓
RoleDataRef

简单单字段访问
    ↓
list[i]

Profiler确认的热点循环
    ↓
Direct Column
```

---

# 📈 性能测试

测试规模：

```text
EntityCount : 500000
SampleCount : 15
WarmupCount : 3
```

以下为 EasyECS 开发阶段的一组性能测试结果。

单位：

```text
ns / entity
```

## Unsafe Backend

| 场景 | List<RoleData> | RoleData[] | ECS list[i] | ECS Ref | ECS Direct |
|---|---:|---:|---:|---:|---:|
| 1 个字段 | 7.946 ns | 0.738 ns | 0.259 ns | 0.309 ns | **0.174 ns** |
| 2 个字段 | 7.520 ns | 0.554 ns | 0.368 ns | 0.378 ns | **0.224 ns** |
| 4 个字段 | 4.213 ns | 0.768 ns | 0.992 ns | 0.799 ns | **0.627 ns** |

## SafeSpan Backend

| 场景 | ECS Ref | ECS Direct |
|---|---:|---:|
| 1 个字段 | 0.534 ns | **0.257 ns** |
| 2 个字段 | 0.739 ns | **0.458 ns** |
| 4 个字段 | 1.531 ns | **0.947 ns** |

> 不同 CPU、Unity 版本、Mono / IL2CPP、平台和编译环境都会影响绝对性能，以上数据主要用于观察不同数据布局和访问方式之间的相对差异。

EasyECS 并不是在宣称：

```text
SoA 一定比普通数组快
```

`RoleData[]` 本身就是非常高效的连续内存结构。

EasyECS 真正解决的是：

> **当 Struct 较大，而热点循环只频繁访问其中少数字段时，避免大量无关数据一起进入 CPU Cache。**

---

# 🎯 适用场景

EasyECS 比较适合：

* 大量同构数据
* 每帧持续遍历
* 单次循环只访问部分字段
* 数万、数十万甚至更多数据
* 对 CPU Cache 较敏感的热点逻辑
* 已经存在大量 OOP 代码的项目
* 不希望整体迁移到 DOTS / 完整 ECS 的项目
* 希望针对热点逐步进行 SoA 优化的项目

典型场景：

```text
角色运行时数据
怪物状态
子弹 / 飞行物
伤害计算
位置 / 速度
Buff运行时状态
AI运行状态
战斗模拟
大规模单位数据
```

---

## 不建议使用的场景

以下情况通常没有必要使用 EasyECS：

* 数据量很少
* 很少进行连续遍历
* 每次基本都会读取 Struct 的全部字段
* 数据主要由 managed reference 组成
* 强依赖稳定对象身份
* 结构操作远多于批量数据处理
* Profiler 中并不存在对应热点

EasyECS 并不是为了：

```text
把项目中的所有 List<T> 全部替换
```

而是：

> **只优化真正值得优化的数据。**

---

# ✨ 核心特点

| 功能 | 状态 |
|---|---|
| OOP 风格访问 | ✅ |
| Source Generator 自动生成 | ✅ |
| `[ECS] / [NotECS]` | ✅ |
| SoA / AoS 混合布局 | ✅ |
| Unsafe Backend | ✅ |
| SafeSpan Backend | ✅ |
| SafeRegistry Backend | ✅ |
| RoleDataRef | ✅ |
| Direct Column | ✅ |
| Resize 后 Ref 保持有效 | ✅ |
| Editor 越界检测 | ✅ |
| Editor 生命周期检测 | ✅ |
| 遗漏 Dispose 检测 | ✅ |
| Finalizer 兜底 | ✅ |
| Benchmark Sample | ✅ |
| UPM Git 安装 | ✅ |

---

# 🧪 Benchmark Sample

EasyECS Package 自带测试用例。

安装插件以后可以通过菜单：

```text
EasyECS
    Import Benchmark Sample
```

也可以在 Package Manager 中：

```text
EasyECS
    Samples
        Benchmark
            Import
```

导入后会得到：

```text
RoleData.cs
RoleDataBenchmark.cs
```

用于测试：

* Add / Get / Set
* Resize
* RoleDataRef
* Direct Column
* Clear
* RemoveAtSwapBack
* 多 ECSList 隔离
* 大量扩容
* OOP 行为一致性
* Dispose
* Editor 生命周期检查
* Player 性能 Benchmark

---

# ✅ 当前测试结果

Unity Correctness Test：

```text
[PASS] Add/Get/Resize
[PASS] Set
[PASS] RoleDataRef
[PASS] Resize后RoleDataRef
[PASS] Direct Column
[PASS] Clear后重新使用
[PASS] RemoveAtSwapBack
[PASS] 多ECSList隔离
[PASS] 大量扩容
[PASS] 混合操作OOP一致性
[PASS] 重复Dispose

[PASS] Unsafe Editor List越界检测
[PASS] Unsafe Editor Dispose后List检测
[PASS] Unsafe Editor Dispose后Ref检测
[PASS] Unsafe Editor Dispose后Column检测
[PASS] Unsafe Editor Clear后Ref检测
[PASS] Unsafe Editor Remove后Ref检测
[PASS] Unsafe Editor SwapBack移动Ref检测
[PASS] Unsafe Editor Remove无关Ref保持有效
[PASS] Unsafe Editor Resize后Ref保持有效
[PASS] Unsafe Editor Add后Column失效
[PASS] Unsafe Editor Remove后Column失效
[PASS] Unsafe Editor Column越界检测
```

结果：

```text
23 / 23 PASS
```

Source Generator 另外还有独立 Roslyn Regression Test：

```text
Total: 21
Pass : 21
Fail : 0
```

覆盖：

```text
布局生成
Backend选择
Managed字段Fallback
标识符转义
代码生成格式
ECS001 ~ ECS004
```

---

# 🆚 与完整 ECS / DOTS 的区别

EasyECS **不是完整 ECS Framework**。

| 项目 | EasyECS | ECS / DOTS |
|---|---|---|
| 数据布局优化 | ✅ | ✅ |
| OOP 业务代码 | ✅ 保留 | 通常需要较大改造 |
| Entity | ❌ | ✅ |
| Component | ❌ | ✅ |
| System | ❌ | ✅ |
| Archetype | ❌ | 常见 |
| Query | ❌ | ✅ |
| Scheduler | ❌ | 常见 |
| Source Generator | ✅ | 不一定 |
| 渐进式接入 | ✅ | 相对困难 |
| 老项目改造成本 | 较低 | 较高 |

EasyECS 更准确的定位是：

```text
OOP-compatible SoA Data Layout Optimizer
```

而不是：

```text
Full ECS Framework
```

---

# 🧩 `[ECS]` 与 `[NotECS]`

Struct 标记：

```csharp
[ECS]
```

后，字段默认按照 SoA 存储：

```csharp
[ECS]
public struct RoleData
{
	public int mHP;
	public float mSpeed;
	[NotECS] public int mID;
}
```

最终布局：

```text
mHP[]
mSpeed[]

AoS[]
    mID
```

也可以反过来：

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

规则：

| Struct | Field | 最终布局 |
|---|---|---|
| `[ECS]` | 无 | SoA |
| `[ECS]` | `[NotECS]` | AoS |
| `[NotECS]` | 无 | AoS |
| `[NotECS]` | `[ECS]` | SoA |

字段 Attribute 会覆盖 Struct 默认设置。

---

# ⚙ 自动 Backend

EasyECS 根据编译环境自动选择 Backend：

```text
AllowUnsafe=true
+
Struct为Unmanaged
        ↓
      Unsafe

否则
        ↓
Span<T>可用
        ↓
     SafeSpan

否则
        ↓
   SafeRegistry
```

查看当前 Backend：

```csharp
Debug.Log(RoleDataECSList.BackendName);
Debug.Log(RoleDataECSList.BackendReason);
```

---

## Unsafe

特点：

* 连续 Native Memory
* 64 Byte 对齐
* 直接 Pointer 访问
* Storage 地址稳定
* Resize 后 Ref 仍然有效
* Direct Column 直接保存字段指针

Unsafe Backend 使用 Native Memory，因此应该主动：

```csharp
Dispose();
```

---

## SafeSpan

SafeSpan 使用托管数组：

```text
HP[]
Speed[]
PositionX[]
PositionY[]
```

不使用：

```text
Marshal.AllocHGlobal
```

因此不会产生 Unsafe Native Memory 遗漏释放的问题。

---

## SafeRegistry

SafeRegistry 通过：

```text
StorageID
    ↓
Static Registry
    ↓
Storage
```

访问数据。

主要作为兼容性 fallback，而不是最高性能 Backend。

---

# 🔒 Ref 与 Column 生命周期

## Ref

普通 Ref：

```csharp
RoleDataRef role = list[i];
```

在 Resize 后仍然有效。

但是：

```text
Clear
RemoveAtSwapBack影响到该元素
Dispose
```

都会使对应 Ref 失效。

---

## Direct Column

Direct Column 为了获得最低访问成本，会直接保存最终数组 / 指针。

因此：

```text
Add
Resize
RemoveAtSwapBack
Clear
Dispose
```

之后旧 Column 都不能继续使用。

需要时重新：

```csharp
var hp = list.getHPColumn();
```

即可。

---

# 🛡 Editor 安全检测

EasyECS 的原则是：

> **Editor 强检查，Player 不为检查付费。**

Editor 中会检测：

* List 越界
* Column 越界
* Dispose 后访问
* Clear 后旧 Ref
* Remove 后旧 Ref
* SwapBack 后失效 Ref
* Add / Remove 后旧 Column
* 遗漏 Dispose

这些检查通过：

```csharp
#if UNITY_EDITOR
#endif
```

存在。

Player 中会被裁掉，不进入字段访问热路径。

因此：

> **Editor 性能 Benchmark 不代表最终 Player 性能。**

---

# 🧯 遗漏 Dispose 检测

Unsafe Backend 使用 Native Memory。

EasyECS 在 Editor 中会自动记录 ECSList 生命周期：

```text
new ECSList
    ↓
自动登记

Dispose
    ↓
自动注销
```

如果忘记 Dispose：

```text
遗漏Dispose
    ↓
自动检测
    ↓
输出错误
    ↓
打印创建堆栈
```

同时 Finalizer 会作为最后一道兜底释放机制。

正常代码仍然推荐：

```csharp
using (RoleDataECSList list = new RoleDataECSList())
{
	...
}
```

---

# 🧪 编译期诊断

Source Generator 会对部分错误直接产生编译诊断：

| Diagnostic | 含义 |
|---|---|
| `ECS001` | 同时标记 `[ECS]` 和 `[NotECS]` |
| `ECS002` | 不支持的数据类型 |
| `ECS003` | 不支持的字段 |
| `ECS004` | 自动生成的 Column API 命名冲突 |

原则是：

> **可以在编译阶段发现的问题，不拖到运行时。**

---

# 📁 源码目录

EasyECS 源码位于 MyFramework：

```text
https://github.com/ZHOURUIH/MyFramework
```

目录：

```text
Packages/com.zhourui.easyecs/
│
├─ Analyzers/
│  └─ ECSGenerator.dll
│
├─ Editor/
│  ├─ EasyECSMenu.cs
│  └─ EasyECS.Editor.asmdef
│
├─ Runtime/
│  ├─ ECSAttribute.cs
│  └─ EasyECS.Runtime.asmdef
│
├─ Samples~/
│  └─ Benchmark/
│
├─ SourceGenerator~/
│  ├─ ECSGenerator.sln
│  ├─ ECSGenerator/
│  └─ ECSGeneratorTest/
│
└─ package.json
```

其中：

```text
Runtime
    → Attribute等运行时接口

Analyzers
    → 编译好的Source Generator

SourceGenerator~
    → Generator完整源码

Samples~
    → Benchmark测试代码
```

---

# 🧭 设计思路

EasyECS 主要遵循以下原则。

### 1. 保留 OOP

不为了数据布局优化而强迫整个业务架构重写。

### 2. 渐进式优化

推荐流程：

```text
Profiler发现热点
        ↓
找到热点Struct
        ↓
添加[ECS]
        ↓
使用ECSList
        ↓
再次测试
```

### 3. Editor 查错，Player 不付费

安全检测尽量留在开发阶段。

### 4. 不为极端情况拖慢正常路径

例如 EasyECS 不会为了让所有 Ref 在任意删除、移动后都保持永久稳定，而给 Player 热路径增加：

```text
Handle
Generation
Dictionary
IndexMap
```

Ref 就是数据 View。

如果业务需要稳定 Entity ID，应由业务层自己维护。

---

# ⚠ 当前限制

目前会主动限制部分复杂声明，例如：

```text
Nested Struct
Generic Struct
ref struct数据定义
Instance Property
readonly Field
fixed Field
```

另外：

* Ref 不是稳定 Entity Handle
* `RemoveAtSwapBack` 会改变元素位置
* Direct Column 不能长期跨结构修改保存
* Unsafe ECSList 应主动 Dispose
* ECSList 的 Resize / Remove / Dispose 当前不设计为线程安全结构操作
* Managed 字段可能导致 Backend 自动切换

这些限制主要是为了避免给正常热路径增加额外运行时成本。

---

# 🔗 相关项目

### EasyECS 展示仓库

```text
https://github.com/ZHOURUIH/EasyECS
```

> 当前仓库不是源码维护仓库。

### EasyECS 源码 / MyFramework

```text
https://github.com/ZHOURUIH/MyFramework
```

源码目录：

```text
Packages/com.zhourui.easyecs
```

### MyServerFramework

```text
https://github.com/ZHOURUIH/MyServerFramework
```

---

# 📄 License

请以源码仓库中的 License 为准。
