# ⚡ EasyECS

一个面向 Unity 的 **OOP 兼容 SoA（Structure of Arrays）数据布局优化插件**。

EasyECS 并不是一套完整的 ECS Framework，也不是 Unity Entities / DOTS 的替代品。

它的目标是：

> **尽量保持原有 OOP 项目的代码结构和开发习惯，只改变热点数据的底层存储布局，以较低的改造成本获得 SoA 更好的 CPU Cache 局部性和连续内存访问性能。**

对于需要优化的数据，只需要添加 `[ECS]` / `[NotECS]`，EasyECS 会通过 **C# Source Generator** 自动生成 SoA Storage、Ref、ECSList、ECSDictionary、Direct Column 等代码。

EasyECS 现在支持 **Hybrid Storage**：同一个 Struct 中可以同时存在 Native SoA、Managed SoA 和 Managed AoS。`string`、`object`、数组、class 引用等 managed 字段不会再让整个 Struct 失去 Unsafe Backend；只有这些字段自身不会进入 Native Memory。

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
RoleDataECSList list = new RoleDataECSList(1024);

list.Add(new RoleData
{
	mHP = 100,
	mSpeed = 5.0f,
	mID = 1,
});

RoleDataRef role = list[0];
role.mHP -= 10;
role.mPositionX += role.mSpeed;
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

# 🗂 ECSDictionary

EasyECS 也会为数据结构生成：

```csharp
RoleDataECSDictionary<TKey>
```

内部结构为：

```text
Dictionary<TKey,int>
        ↓
    dense index
        ↓
RoleDataECSList
```

因此随机 Key 查询仍由 BCL `Dictionary<TKey,int>` 负责，而 Value 使用 EasyECS 的连续存储。

典型写法：

```csharp
RoleDataECSDictionary<int> roles = new RoleDataECSDictionary<int>();

roles.Add(1001, new RoleData
{
	mHP = 100,
	mSpeed = 5.0f,
});

roles[1001].mHP -= 10;

RoleDataRef role = roles[1001];
role.mPositionX += role.mSpeed;
```

支持的常用接口包括：

```text
Add
TryAdd
ContainsKey
TryGetValue
TryGetIndex
Remove
Clear
Count
Capacity
Comparer
getKeyAt
getValueAt
foreach
Keys
Values
Direct Column
Dispose
```

`Remove` 使用 dense swap-back，因此遍历顺序不保证稳定。

对于连续批处理，同样可以使用 Direct Column：

```csharp
var hp = roles.getHPColumn();

for (int i = 0; i < roles.Count; ++i)
{
	hp[i] -= 1;
}
```

---

# 📈 性能测试

以下数据全部来自实际运行日志。

> 当前表格记录每个 Benchmark 测试项的 **Median 耗时 / 单位操作耗时**。数值越低越好。

以下结果全部来自实际运行日志。为了让 README 可读且便于横向比较，当前结果表记录 **每一个 Benchmark 测试项的 Median 耗时与单位操作耗时**，格式统一为：

```text
Median ms / ns per entity(or op)
```

数值越低越好。每轮当前 Benchmark 使用：

```text
EntityCount      = 500000
SampleCount      = 15
WarmupCount      = 3
RandomWriteCount = 50000
```

`RandomWriteCount` 仅用于 Dictionary 随机修改测试。原始日志中的 Min / Max 用于观察采样抖动，README 不把它们作为最终性能排序指标，因此下面统一使用 Median。

### PC 测试环境

```text
Unity       : 6000.3.21f1
Platform    : Windows x64 Player
Graphics    : Direct3D 12
GPU         : NVIDIA GeForce RTX 2060
CPU threads : 32
```

#### PC 当前 backend-agnostic Benchmark：SafeSpan / SafeRegistry

SafeSpan 与 SafeRegistry 均使用当前 backend-agnostic Benchmark，同一套业务测试代码不包含 backend-specific pointer 访问。

#### List：修改 1 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `List<RoleData>` | 4.035 / 8.069 | 3.837 / 7.675 |
| `RoleData[]` | 0.323 / 0.646 | 0.297 / 0.595 |
| `ECS list[i]` | 0.358 / 0.716 | 0.815 / 1.630 |
| `ECS Ref` | 0.370 / 0.741 | 0.810 / 1.619 |
| `ECS Direct` | 0.176 / 0.352 | 0.183 / 0.366 |

#### List：访问 2 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `List<RoleData>` | 3.858 / 7.715 | 4.011 / 8.022 |
| `RoleData[]` | 0.313 / 0.626 | 0.306 / 0.612 |
| `ECS list[i]` | 0.637 / 1.274 | 1.468 / 2.935 |
| `ECS Ref` | 0.449 / 0.898 | 1.410 / 2.821 |
| `ECS Direct` | 0.220 / 0.441 | 0.232 / 0.465 |

#### List：访问 4 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `List<RoleData>` | 2.141 / 4.282 | 2.224 / 4.448 |
| `RoleData[]` | 0.503 / 1.007 | 0.472 / 0.944 |
| `ECS list[i]` | 1.478 / 2.955 | 3.808 / 7.615 |
| `ECS Ref` | 0.982 / 1.964 | 3.504 / 7.007 |
| `ECS Direct` | 0.582 / 1.164 | 0.502 / 1.003 |

#### Dictionary：随机 Key 读取

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 14.235 / 28.470 | 13.822 / 27.644 |
| `IndexMap + RoleData[]` | 21.080 / 42.160 | 20.982 / 41.963 |
| `IndexMap + int[]` | 7.639 / 15.279 | 7.741 / 15.483 |
| `ECS Inline Indexer` | 8.736 / 17.472 | 13.122 / 26.245 |
| `ECS Local Ref` | 8.898 / 17.795 | 14.050 / 28.100 |
| `ECS TryGetValue` | 8.585 / 17.170 | 11.511 / 23.023 |

#### Dictionary：随机 Key 修改

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 1.162 / 23.240 | 1.364 / 27.286 |
| `IndexMap + RoleData[]` | 0.805 / 16.102 | 0.962 / 19.242 |
| `IndexMap + int[]` | 0.701 / 14.014 | 0.730 / 14.598 |
| `ECS Inline Indexer` | 0.781 / 15.614 | 1.047 / 20.936 |
| `ECS Local Ref` | 0.783 / 15.654 | 1.044 / 20.888 |
| `ECS TryGetValue` | 0.791 / 15.816 | 0.826 / 16.510 |

#### Dictionary：连续存储全量修改 1 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `Dictionary Key全量更新` | 10.515 / 21.030 | 10.419 / 20.839 |
| `Dense RoleData[]` | 0.278 / 0.556 | 0.284 / 0.568 |
| `Dense int[]` | 0.143 / 0.285 | 0.151 / 0.301 |
| `ECS Dense Ref` | 0.447 / 0.894 | 0.859 / 1.718 |
| `ECS Direct` | 0.235 / 0.470 | 0.243 / 0.485 |

#### Dictionary：连续存储全量访问 4 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `Dictionary Key全量更新` | 6.679 / 13.359 | 6.843 / 13.687 |
| `Dense RoleData[]` | 0.528 / 1.056 | 0.459 / 0.917 |
| `ECS Dense Ref` | 1.087 / 2.173 | 3.734 / 7.468 |
| `ECS Direct` | 0.581 / 1.161 | 0.590 / 1.180 |

#### Dictionary：Dense 全量更新 + 10% 随机 Key 修改

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `Dictionary<int,RoleData>` | 14.203 / 25.824 | 14.242 / 25.894 |
| `IndexMap + RoleData[]` | 1.733 / 3.151 | 1.277 / 2.322 |
| `ECS Direct+LocalRef` | 1.094 / 1.989 | 1.147 / 2.085 |

#### Dictionary：仅遍历 Key

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `for + getKeyAt` | 0.189 / 0.378 | 0.177 / 0.354 |
| `foreach dict + item.Key` | 0.230 / 0.460 | 0.212 / 0.423 |
| `foreach dict.Keys` | 1.909 / 3.818 | 1.880 / 3.760 |

#### Dictionary：仅遍历 Value 并修改 1 个字段

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `for + getValueAt` | 0.470 / 0.939 | 0.857 / 1.714 |
| `foreach dict + item.Value` | 0.421 / 0.841 | 0.812 / 1.623 |
| `foreach dict.Values` | 1.825 / 3.649 | 1.841 / 3.681 |
| `Direct Column` | 0.235 / 0.470 | 0.227 / 0.455 |

#### Dictionary：同时遍历 Key + Value

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `for getKeyAt+getValueAt` | 0.538 / 1.075 | 0.954 / 1.907 |
| `foreach dict` | 0.400 / 0.800 | 0.855 / 1.709 |

#### Enumerator：仅读取 Key

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `int[] for` | 0.123 / 0.246 | 0.117 / 0.235 |
| `int[] foreach` | 0.123 / 0.247 | 0.131 / 0.263 |
| `ECS for + getKeyAt` | 0.196 / 0.391 | 0.187 / 0.373 |
| `ECS foreach dict + item.Key` | 0.215 / 0.430 | 0.220 / 0.440 |
| `ECS foreach dict.Keys` | 1.848 / 3.696 | 1.876 / 3.752 |
| `ECS Keys手动Enumerator` | 1.849 / 3.698 | 1.853 / 3.706 |
| `Dictionary foreach` | 3.802 / 7.604 | 4.044 / 8.088 |
| `Dictionary foreach Keys` | 0.547 / 1.094 | 0.688 / 1.377 |
| `Dictionary Keys手动Enumerator` | 0.495 / 0.989 | 0.448 / 0.896 |

#### Enumerator：仅读取 Value.mHP

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `RoleData[] for` | 0.205 / 0.410 | 0.216 / 0.432 |
| `RoleData[] foreach` | 0.216 / 0.432 | 0.211 / 0.421 |
| `ECS for + getValueAt` | 0.450 / 0.900 | 0.871 / 1.742 |
| `ECS foreach dict + Value` | 0.403 / 0.806 | 0.846 / 1.691 |
| `ECS foreach dict.Values` | 1.801 / 3.603 | 1.817 / 3.634 |
| `ECS Values手动Enumerator` | 1.804 / 3.609 | 1.816 / 3.631 |
| `Dictionary foreach` | 3.776 / 7.551 | 3.793 / 7.586 |
| `Dictionary foreach Values` | 0.827 / 1.655 | 0.781 / 1.562 |
| `Dictionary Values手动Enumerator` | 0.431 / 0.861 | 0.399 / 0.799 |

#### Enumerator：修改 Value.mHP

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `ECS for + getValueAt` | 0.449 / 0.898 | 0.852 / 1.703 |
| `ECS foreach dict + Value` | 0.398 / 0.795 | 0.839 / 1.678 |
| `ECS foreach dict.Values` | 1.790 / 3.580 | 1.843 / 3.686 |
| `ECS Values手动Enumerator` | 1.792 / 3.584 | 1.840 / 3.680 |
| `ECS Direct Column` | 0.235 / 0.469 | 0.235 / 0.469 |

#### Enumerator：同时读取 Key + Value

| 测试项 | SafeSpan | SafeRegistry |
|---|---:|---:|
| `ECS for getKeyAt+getValueAt` | 0.539 / 1.078 | 0.973 / 1.947 |
| `ECS foreach dict` | 0.415 / 0.830 | 0.825 / 1.650 |
| `ECS 手动Enumerator` | 0.401 / 0.801 | 0.805 / 1.610 |
| `Dictionary foreach` | 3.835 / 7.671 | 3.828 / 7.656 |
| `Dictionary 手动Enumerator` | 0.531 / 1.062 | 0.575 / 1.151 |


#### PC Unsafe 历史 Benchmark

PC Unsafe 是在较早一次 Generator / Benchmark 修订上测得的完整结果。它已经覆盖 List、Dictionary、Dense、Mixed 与 Enumerator，但当时随机 Dictionary Benchmark 仍额外包含 `IndexMap + int*` 研究基线，而且部分 Dictionary 微基准实现随后又做过调整。

因此下面结果保留作为 **Unsafe 在 PC 上的历史实测数据**，但不应拿它与上面的当前 SafeSpan / SafeRegistry 做严格微秒级横向排名。当前正式 Sample 已移除业务代码中的 `int*` backend-specific 基线。

旧版随机修改微基准的 `ns/op` 归一化口径也与当前版本不同，因此本节只保留最可靠的 **Median ms**，不展示旧版 `ns/op`。

#### List：修改 1 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `List<RoleData>` | 3.995 |
| `RoleData[]` | 0.292 |
| `ECS list[i]` | 0.274 |
| `ECS Ref` | 0.271 |
| `ECS Direct` | 0.078 |

#### List：访问 2 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `List<RoleData>` | 3.773 |
| `RoleData[]` | 0.317 |
| `ECS list[i]` | 0.207 |
| `ECS Ref` | 0.219 |
| `ECS Direct` | 0.098 |

#### List：访问 4 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `List<RoleData>` | 2.124 |
| `RoleData[]` | 0.461 |
| `ECS list[i]` | 0.819 |
| `ECS Ref` | 0.535 |
| `ECS Direct` | 0.326 |

#### Dictionary：随机 Key 读取

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `Dictionary<int,RoleData>` | 11.993 |
| `IndexMap + RoleData[]` | 16.519 |
| `IndexMap + int[]` | 6.731 |
| `IndexMap + int*` | 6.758 |
| `ECS Inline Indexer` | 7.470 |
| `ECS Local Ref` | 7.642 |
| `ECS TryGetValue` | 7.511 |

#### Dictionary：随机 Key 修改

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `Dictionary<int,RoleData>` | 24.653 |
| `IndexMap + RoleData[]` | 17.320 |
| `IndexMap + int[]` | 7.226 |
| `IndexMap + int*` | 7.156 |
| `ECS Inline Indexer` | 7.792 |
| `ECS Local Ref` | 8.051 |
| `ECS TryGetValue` | 10.206 |

#### Dictionary：连续存储全量修改 1 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `Dictionary Key全量更新` | 10.338 |
| `Dense RoleData[]` | 0.313 |
| `Dense int[]` | 0.156 |
| `ECS Dense Ref` | 0.188 |
| `ECS Direct` | 0.091 |

#### Dictionary：连续存储全量访问 4 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `Dictionary Key全量更新` | 6.888 |
| `Dense RoleData[]` | 0.450 |
| `ECS Dense Ref` | 0.500 |
| `ECS Direct` | 0.321 |

#### Dictionary：Dense 全量更新 + 10% 随机 Key 修改

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `Dictionary<int,RoleData>` | 9.780 |
| `IndexMap + RoleData[]` | 1.949 |
| `ECS Direct+LocalRef` | 1.101 |

#### Dictionary：仅遍历 Key

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `for + getKeyAt` | 0.156 |
| `foreach dict + item.Key` | 0.238 |
| `foreach dict.Keys` | 1.869 |

#### Dictionary：仅遍历 Value 并修改 1 个字段

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `for + getValueAt` | 0.176 |
| `foreach dict + item.Value` | 0.245 |
| `foreach dict.Values` | 1.811 |
| `Direct Column` | 0.078 |

#### Dictionary：同时遍历 Key + Value

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `for getKeyAt+getValueAt` | 0.224 |
| `foreach dict` | 0.275 |

#### Enumerator：仅读取 Key

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `int[] for` | 0.118 |
| `int[] foreach` | 0.121 |
| `ECS for + getKeyAt` | 0.133 |
| `ECS foreach dict + item.Key` | 0.220 |
| `ECS foreach dict.Keys` | 1.855 |
| `ECS Keys手动Enumerator` | 1.834 |
| `Dictionary foreach` | 3.788 |
| `Dictionary foreach Keys` | 0.945 |
| `Dictionary Keys手动Enumerator` | 0.500 |

#### Enumerator：仅读取 Value.mHP

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `RoleData[] for` | 0.229 |
| `RoleData[] foreach` | 0.230 |
| `ECS for + getValueAt` | 0.156 |
| `ECS foreach dict + Value` | 0.243 |
| `ECS foreach dict.Values` | 1.864 |
| `ECS Values手动Enumerator` | 1.829 |
| `Dictionary foreach` | 3.782 |
| `Dictionary foreach Values` | 0.859 |
| `Dictionary Values手动Enumerator` | 0.424 |

#### Enumerator：修改 Value.mHP

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `ECS for + getValueAt` | 0.173 |
| `ECS foreach dict + Value` | 0.239 |
| `ECS foreach dict.Values` | 1.810 |
| `ECS Values手动Enumerator` | 1.793 |
| `ECS Direct Column` | 0.078 |

#### Enumerator：同时读取 Key + Value

| 测试项 | Unsafe（历史 Median ms） |
|---|---:|
| `ECS for getKeyAt+getValueAt` | 0.220 |
| `ECS foreach dict` | 0.268 |
| `ECS 手动Enumerator` | 0.266 |
| `Dictionary foreach` | 3.828 |
| `Dictionary 手动Enumerator` | 0.556 |


### Android 真机测试环境

```text
Device            : HUAWEI ALP-AL00
OS                : Android 10 / API 29
CPU               : ARM64, 8 Cores
big.LITTLE        : 4 big + 4 little
Memory            : 3648 MB
Unity             : 6000.3.21f1
Build Type        : Release
Scripting Backend : IL2CPP
CPU Target        : arm64-v8a
Code Stripping    : Enabled
```

Android 的 Unsafe、SafeSpan、SafeRegistry 均在同一台真机、同一套当前 backend-agnostic Benchmark 上运行。

三轮测试为连续执行，手机温度、动态调频与大小核调度会影响绝对耗时。因此这些数据适合观察数量级、访问模式和 Backend 的结构性差异，不应把非常小的毫秒差当成跨设备固定结论。

### Android Benchmark

#### List：修改 1 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `List<RoleData>` | 12.610 / 25.221 | 13.157 / 26.315 | 12.602 / 25.204 |
| `RoleData[]` | 3.263 / 6.526 | 3.524 / 7.048 | 3.285 / 6.571 |
| `ECS list[i]` | 1.063 / 2.126 | 2.305 / 4.609 | 6.713 / 13.425 |
| `ECS Ref` | 1.063 / 2.126 | 2.343 / 4.685 | 6.563 / 13.126 |
| `ECS Direct` | 0.750 / 1.500 | 1.068 / 2.135 | 1.166 / 2.332 |

#### List：访问 2 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `List<RoleData>` | 12.436 / 24.871 | 12.532 / 25.065 | 12.612 / 25.224 |
| `RoleData[]` | 3.257 / 6.513 | 3.408 / 6.817 | 3.301 / 6.602 |
| `ECS list[i]` | 1.278 / 2.555 | 3.785 / 7.570 | 13.137 / 26.274 |
| `ECS Ref` | 1.317 / 2.633 | 2.905 / 5.810 | 12.673 / 25.346 |
| `ECS Direct` | 0.940 / 1.879 | 1.390 / 2.780 | 1.595 / 3.190 |

#### List：访问 4 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `List<RoleData>` | 12.189 / 24.377 | 12.243 / 24.485 | 12.298 / 24.597 |
| `RoleData[]` | 4.256 / 8.513 | 2.934 / 5.869 | 4.321 / 8.642 |
| `ECS list[i]` | 2.758 / 5.516 | 8.879 / 17.757 | 32.704 / 65.408 |
| `ECS Ref` | 2.743 / 5.486 | 4.802 / 9.603 | 30.067 / 60.133 |
| `ECS Direct` | 2.683 / 5.367 | 3.380 / 6.759 | 3.381 / 6.761 |

#### Dictionary：随机 Key 读取

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `Dictionary<int,RoleData>` | 210.057 / 420.115 | 210.311 / 420.622 | 211.278 / 422.556 |
| `IndexMap + RoleData[]` | 211.423 / 422.846 | 235.051 / 470.102 | 216.365 / 432.730 |
| `IndexMap + int[]` | 204.765 / 409.529 | 209.595 / 419.190 | 206.285 / 412.571 |
| `ECS Inline Indexer` | 210.568 / 421.135 | 208.101 / 416.201 | 218.437 / 436.874 |
| `ECS Local Ref` | 211.314 / 422.628 | 205.736 / 411.471 | 214.262 / 428.523 |
| `ECS TryGetValue` | 209.584 / 419.168 | 210.138 / 420.276 | 214.062 / 428.124 |

#### Dictionary：随机 Key 修改

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `Dictionary<int,RoleData>` | 24.967 / 499.344 | 24.878 / 497.562 | 25.123 / 502.468 |
| `IndexMap + RoleData[]` | 31.220 / 624.396 | 31.136 / 622.728 | 30.819 / 616.386 |
| `IndexMap + int[]` | 27.283 / 545.656 | 27.256 / 545.126 | 27.388 / 547.750 |
| `ECS Inline Indexer` | 27.467 / 549.334 | 27.583 / 551.656 | 28.051 / 561.010 |
| `ECS Local Ref` | 27.359 / 547.188 | 27.402 / 548.042 | 28.213 / 564.260 |
| `ECS TryGetValue` | 27.295 / 545.906 | 27.661 / 553.228 | 28.098 / 561.958 |

#### Dictionary：连续存储全量修改 1 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `Dictionary Key全量更新` | 51.985 / 103.970 | 52.605 / 105.209 | 51.838 / 103.675 |
| `Dense RoleData[]` | 3.236 / 6.472 | 3.232 / 6.464 | 3.622 / 7.244 |
| `Dense int[]` | 0.743 / 1.485 | 0.743 / 1.486 | 0.742 / 1.483 |
| `ECS Dense Ref` | 1.062 / 2.124 | 2.486 / 4.972 | 6.725 / 13.449 |
| `ECS Direct` | 0.748 / 1.496 | 0.955 / 1.909 | 1.167 / 2.334 |

#### Dictionary：连续存储全量访问 4 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `Dictionary Key全量更新` | 52.557 / 105.114 | 52.583 / 105.166 | 52.134 / 104.268 |
| `Dense RoleData[]` | 3.321 / 6.642 | 3.315 / 6.629 | 3.808 / 7.617 |
| `ECS Dense Ref` | 2.420 / 4.841 | 5.150 / 10.300 | 30.271 / 60.543 |
| `ECS Direct` | 2.663 / 5.325 | 3.225 / 6.450 | 3.385 / 6.771 |

#### Dictionary：Dense 全量更新 + 10% 随机 Key 修改

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `Dictionary<int,RoleData>` | 77.504 / 140.917 | 77.764 / 141.388 | 81.010 / 147.291 |
| `IndexMap + RoleData[]` | 34.900 / 63.455 | 34.873 / 63.405 | 34.503 / 62.732 |
| `ECS Direct+LocalRef` | 28.815 / 52.390 | 29.384 / 53.426 | 30.138 / 54.795 |

#### Dictionary：仅遍历 Key

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `for + getKeyAt` | 0.954 / 1.908 | 0.954 / 1.908 | 0.955 / 1.910 |
| `foreach dict + item.Key` | 1.591 / 3.181 | 1.591 / 3.181 | 1.591 / 3.181 |
| `foreach dict.Keys` | 1.270 / 2.541 | 1.269 / 2.539 | 1.239 / 2.478 |

#### Dictionary：仅遍历 Value 并修改 1 个字段

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `for + getValueAt` | 1.061 / 2.122 | 2.478 / 4.955 | 6.761 / 13.522 |
| `foreach dict + item.Value` | 1.803 / 3.606 | 2.904 / 5.807 | 8.849 / 17.699 |
| `foreach dict.Values` | 1.107 / 2.214 | 2.468 / 4.936 | 6.678 / 13.356 |
| `Direct Column` | 0.766 / 1.531 | 0.955 / 1.909 | 1.167 / 2.334 |

#### Dictionary：同时遍历 Key + Value

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `for getKeyAt+getValueAt` | 1.617 / 3.233 | 3.111 / 6.223 | 7.521 / 15.042 |
| `foreach dict` | 2.067 / 4.133 | 3.133 / 6.266 | 8.078 / 16.156 |

#### Enumerator：仅读取 Key

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `int[] for` | 0.639 / 1.277 | 0.638 / 1.275 | 0.640 / 1.279 |
| `int[] foreach` | 0.638 / 1.276 | 0.639 / 1.278 | 0.639 / 1.278 |
| `ECS for + getKeyAt` | 0.956 / 1.912 | 0.957 / 1.915 | 0.956 / 1.913 |
| `ECS foreach dict + item.Key` | 1.591 / 3.182 | 1.591 / 3.182 | 1.592 / 3.184 |
| `ECS foreach dict.Keys` | 1.269 / 2.539 | 1.269 / 2.538 | 1.258 / 2.516 |
| `ECS Keys手动Enumerator` | 1.270 / 2.540 | 1.269 / 2.538 | 1.270 / 2.540 |
| `Dictionary foreach` | 13.611 / 27.222 | 13.647 / 27.294 | 13.795 / 27.591 |
| `Dictionary foreach Keys` | 4.978 / 9.956 | 5.000 / 10.000 | 4.983 / 9.966 |
| `Dictionary Keys手动Enumerator` | 4.945 / 9.891 | 4.950 / 9.899 | 4.922 / 9.845 |

#### Enumerator：仅读取 Value.mHP

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `RoleData[] for` | 2.573 / 5.146 | 2.639 / 5.277 | 2.763 / 5.526 |
| `RoleData[] foreach` | 2.580 / 5.159 | 2.640 / 5.279 | 2.758 / 5.517 |
| `ECS for + getValueAt` | 0.743 / 1.486 | 2.432 / 4.864 | 6.645 / 13.290 |
| `ECS foreach dict + Value` | 1.709 / 3.418 | 2.801 / 5.601 | 8.454 / 16.907 |
| `ECS foreach dict.Values` | 1.079 / 2.158 | 2.347 / 4.694 | 6.597 / 13.194 |
| `ECS Values手动Enumerator` | 1.063 / 2.125 | 2.344 / 4.687 | 6.584 / 13.168 |
| `Dictionary foreach` | 13.543 / 27.085 | 13.548 / 27.096 | 13.805 / 27.609 |
| `Dictionary foreach Values` | 5.713 / 11.425 | 5.623 / 11.246 | 5.548 / 11.097 |
| `Dictionary Values手动Enumerator` | 5.701 / 11.401 | 5.706 / 11.411 | 5.585 / 11.171 |

#### Enumerator：修改 Value.mHP

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `ECS for + getValueAt` | 1.061 / 2.122 | 2.487 / 4.974 | 6.748 / 13.496 |
| `ECS foreach dict + Value` | 1.805 / 3.609 | 2.906 / 5.811 | 8.628 / 17.256 |
| `ECS foreach dict.Values` | 1.103 / 2.205 | 2.469 / 4.939 | 6.679 / 13.357 |
| `ECS Values手动Enumerator` | 1.109 / 2.218 | 2.465 / 4.930 | 6.675 / 13.350 |
| `ECS Direct Column` | 0.781 / 1.563 | 0.954 / 1.908 | 1.168 / 2.335 |

#### Enumerator：同时读取 Key + Value

| 测试项 | Unsafe | SafeSpan | SafeRegistry |
|---|---:|---:|---:|
| `ECS for getKeyAt+getValueAt` | 1.606 / 3.211 | 3.125 / 6.250 | 7.478 / 14.956 |
| `ECS foreach dict` | 2.057 / 4.115 | 3.117 / 6.234 | 8.317 / 16.633 |
| `ECS 手动Enumerator` | 2.053 / 4.106 | 3.126 / 6.251 | 8.332 / 16.664 |
| `Dictionary foreach` | 14.000 / 27.999 | 14.039 / 28.078 | 13.888 / 27.776 |
| `Dictionary 手动Enumerator` | 13.589 / 27.178 | 13.735 / 27.470 | 13.669 / 27.338 |


### Benchmark 结论

从 PC 与 Android 真机结果可以得到几条比较稳定的结论：

- **Unsafe 的 Native Storage 是最高性能路径。** Android ARM64 + IL2CPP 下，List 单字段 `ECS Direct` 为 `0.750 ms`，`ECS Ref` 为 `1.063 ms`；Dictionary 连续存储单字段场景中 `Dense int[]` 为 `0.743 ms`，`ECS Direct` 为 `0.748 ms`，已经非常接近手写连续数组。这里的 Benchmark 数据来自全部字段可使用 Native Storage 的 `RoleData`；新增的 Managed Hybrid Storage 主要解决“少量引用字段拖累整个结构体”的问题，不应直接把这些数字当作 managed 字段访问性能。
- **SafeSpan 是安全路径的性能默认选择。** 相比 SafeRegistry，Ref / indexer 在多字段访问时明显更快，同时 Direct Column 仍然保持较低成本。
- **SafeRegistry 的主要定位是旧运行环境与热更新兼容。** 它的 Ref 在多字段热点中成本明显更高，但 Direct Column 与 SafeSpan 很接近，因此兼容模式下仍然可以通过 Direct Column 获得良好的批处理性能。
- **随机 Dictionary 查询的主要成本来自 Hash 查找。** Android 真机随机读取时三个 Backend 的 EasyECS 路径差距远小于 Dense 批处理场景，说明数据布局优化最适合连续、高频访问。
- **热点循环优先使用 Direct Column。** 普通业务逻辑仍建议使用 `Ref` / indexer，只有 Profiler 确认的高频批处理才需要下沉到 Direct Column。

Benchmark 是特定设备、Unity 版本、编译器和运行状态下的测量结果，不代表所有项目都能得到完全相同的倍率。建议通过 `Samples~/Benchmark` 在目标项目和目标设备上重新运行。

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
| Managed / Native Hybrid Storage | ✅ |
| Managed ECS 字段 SoA | ✅ |
| Unsafe Backend | ✅ |
| SafeSpan Backend | ✅ |
| SafeRegistry Backend | ✅ |
| ECSList | ✅ |
| ECSDictionary | ✅ |
| Dictionary Enumerator | ✅ |
| RoleDataRef | ✅ |
| Direct Column | ✅ |
| Resize 后 Ref 保持有效 | ✅ |
| Editor 越界检测 | ✅ |
| Editor 生命周期检测 | ✅ |
| 遗漏 Dispose 检测 | ✅ |
| Finalizer 兜底 | ✅ |
| Benchmark Sample | ✅ |
| UPM Git 安装 | ✅ |

Hybrid Storage 的核心目标是：

```text
热点 unmanaged 字段
→ 尽可能继续使用 Native SoA

managed 字段
→ 保持 GC 正确管理

业务 API
→ 不需要区分底层 Backend
```

---

# 🧪 Benchmark Sample

EasyECS Package 自带完整测试与 Benchmark Sample。

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

导入后主要包含：

```text
RoleData.cs
RoleDataBenchmark.cs
RoleDataDictionaryBenchmark.cs
RoleDataDictionaryEnumeratorBenchmark.cs
EasyECSRuntimeUnitTest.cs
```

覆盖内容包括：

```text
ECSList
├─ Add / Get / Set
├─ Resize
├─ Resize后旧Ref
├─ Direct Column
├─ Clear
├─ RemoveAtSwapBack
└─ Dispose

ECSDictionary
├─ Add / TryAdd
├─ Indexer / TryGetValue
├─ TryGetIndex
├─ Remove / dense swap-back
├─ foreach
├─ Keys / Values
├─ Direct Column
└─ Dispose

Managed / Hybrid
├─ managed ECS字段
├─ managed AoS字段
├─ string / object / null
├─ Resize后旧Ref
├─ Direct Column
├─ RemoveAtSwapBack
├─ Clear
└─ Dictionary路径

Editor
├─ List / Column越界
├─ Dispose后访问
├─ Clear / Remove后的旧Ref
├─ SwapBack移动Ref
├─ 结构变化后旧Column
└─ 遗漏Dispose检测
```

性能部分则覆盖：

```text
List
Dictionary随机访问
Dictionary连续遍历
Dense + Random混合场景
Dictionary foreach
Keys / Values
Enumerator
Direct Column
```

---

# ✅ 当前测试与验证状态

Source Generator Regression Test 已随着 Hybrid Storage 设计扩展为 **42 个测试项**，覆盖重点包括：

```text
布局生成
Backend选择
Unsafe / SafeSpan / SafeRegistry
Managed Hybrid Unsafe
Managed ECS字段
Managed AoS
ECSDictionary Hybrid
Resize提交顺序
构造失败清理
标识符转义
代码生成格式
ECS001 ~ ECS004
```

Runtime Unit Test 覆盖：

```text
ECSList
ECSDictionary
Managed字段
Ref生命周期
Direct Column
RemoveAtSwapBack
Clear
Resize
Dispose
Editor安全检查
```

在 Hybrid Storage 改动之前，Android ARM64 + IL2CPP 已经分别实际跑过：

```text
Unsafe
SafeSpan
SafeRegistry
```

三条 Backend 路径。

当前 Hybrid Storage 是在这轮真机验证之后新增的核心存储路径，因此 **不能把之前的 Android 通过结果直接当成 Hybrid Storage 已验证结果**。

发布包含 Hybrid Storage 的版本前，应重新运行最新：

```text
Generator Test
→ 42项

Unity Runtime Unit Test
→ 重点检查 managed List / Dictionary
→ Resize后旧Ref
→ Direct Column
→ RemoveAtSwapBack
→ Clear
→ Dispose
```

Benchmark 表中的 Unsafe 数据使用的是纯 Native `RoleData`，用于反映 Native Storage 热路径性能；它并不代表 `string` / `object` 等 managed 字段本身的访问性能。

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

后，字段默认按照 ECS/SoA 方式存储：

```csharp
[ECS]
public struct RoleData
{
	public int mHP;
	public float mSpeed;
	[NotECS] public int mID;
}
```

逻辑布局：

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

基础规则：

| Struct | Field | 最终逻辑布局 |
|---|---|---|
| `[ECS]` | 无 | SoA |
| `[ECS]` | `[NotECS]` | AoS |
| `[NotECS]` | 无 | AoS |
| `[NotECS]` | `[ECS]` | SoA |

字段 Attribute 会覆盖 Struct 默认设置。

这里的 `[ECS] / [NotECS]` 决定的是 **SoA / AoS 逻辑布局**，并不等价于：

```text
[ECS]    = Native
[NotECS] = Managed
```

managed 字段仍然遵循 `[ECS] / [NotECS]` 的布局语义。

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
	[NotECS] public string mModelPath;
}
```

开启 Unsafe 后：

```text
[ECS] int/float
→ Native SoA

[ECS] string/object
→ Managed SoA

[NotECS] int + string
→ Managed AoS
```

因此：

```text
managed字段 ≠ 自动[NotECS]
```

`string mName` 如果没有写 `[NotECS]`，它仍然是 ECS 字段，只是物理存储为 `string[]`，而不是 native pointer。

---

# ⚙ 自动 Backend

EasyECS 根据当前 Compilation 自动选择 Backend：

```text
ECS_FORCE_SAFE_REGISTRY
        ↓
   SafeRegistry

否则

Allow Unsafe Code=true
+
存在可放入 Native Storage 的字段
        ↓
      Unsafe

否则

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

常见 `BackendReason`：

```text
AllowUnsafe=true,Unmanaged=true
AllowUnsafe=true,HybridStorage=true
NoNativeStorage,Span=true
ECS_FORCE_SAFE_REGISTRY
SpanUnavailable
```

## Unsafe

Unsafe 是最高性能路径。

对于能够安全放入 Native Memory 的 unmanaged 数据，使用：

```text
Native Memory
+
64 Byte Alignment
+
Pointer
```

但现在 **Unsafe Backend 不再要求整个 Struct 都是 unmanaged**。

Unsafe 支持 Hybrid Storage：

```text
一个Struct
├─ Native SoA
├─ Managed SoA
└─ Native / Managed AoS
```

Storage 地址保持稳定，Resize 后 Ref 可以继续访问同一个 Storage 入口。

## Managed 字段与 Hybrid Storage

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
	[NotECS] public string mModelPath;
}
```

当：

```text
Allow Unsafe Code = true
```

时，物理存储拆分为：

```text
Native SoA
├─ int*   mHP
└─ float* mSpeed

Managed SoA
├─ string[] mName
└─ object[] mPayload

Managed AoS
└─ RoleRuntimeDataAoSBlock[]
   ├─ int mID
   └─ string mModelPath
```

完整规则：

```text
[ECS] + unmanaged
→ Native SoA（Unsafe Backend）

[ECS] + managed
→ Managed SoA

[NotECS] 且AoS Block全部unmanaged
→ Native AoS（Unsafe Backend）

[NotECS] 的AoS Block只要包含managed字段
→ 整个AoS Block使用Managed AoS
```

因此这些类型都可以正常出现在 EasyECS 数据中：

```text
string
object
array
class reference
UnityEngine.Object reference
其他managed reference
```

只是这些字段本身不会进入 Native Memory。

这解决了原先这种问题：

```csharp
[ECS]
public struct RoleData
{
	public int mHP;
	public float mPositionX;
	public float mPositionY;
	public string mName;
}
```

旧的整 Struct 判断会因为 `mName` 是 `string`，让 `mHP / mPositionX / mPositionY` 一起失去 Unsafe。

Hybrid Storage 下：

```text
mHP
mPositionX
mPositionY
→ 继续走Native SoA

mName
→ string[]
```

少量业务引用字段不会再拖累整个热点 Struct。

如果 Struct **完全没有任何可放入 Native Storage 的字段**，即使开启了 Allow Unsafe Code，也不会为了形式上使用 Unsafe 额外创建 Native Storage：

```text
No Native Storage Candidate
        ↓
Span<T>可用
        ↓
SafeSpan

否则
        ↓
SafeRegistry
```

## SafeSpan

SafeSpan 使用托管数组：

```text
HP[]
Speed[]
PositionX[]
PositionY[]
string[]
object[]
```

并通过 Span / 稳定 Storage 入口访问。

它不使用 Native Memory，因此不会产生 Unsafe Native Allocation 的释放问题。

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

主要定位：

```text
旧运行环境
缺少Span<T>
热更新兼容
```

而不是作为最高性能 Backend。

如果需要强制测试 SafeRegistry，在当前 Build Target 的 Scripting Define Symbols 中加入：

```text
ECS_FORCE_SAFE_REGISTRY
```

只要当前 Compilation 中存在该宏，就会强制：

```text
SafeRegistry
```

它的优先级高于 Allow Unsafe Code 和 Span 检测。

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

三个 Backend 都提供统一的 `Dispose()` API。容器生命周期结束时建议主动调用 `Dispose()`：

```csharp
RoleDataECSList list = new RoleDataECSList();
try
{
	...
}
finally
{
	list.Dispose();
}
```

`using` 可以使用，但不是 EasyECS 普通业务代码必须采用的写法。

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

* Ref 不是稳定 Entity Handle。
* `RemoveAtSwapBack` 会改变元素位置。
* Direct Column 不能长期跨结构修改保存。
* Unsafe Backend 中存在 Native Storage 时应主动 `Dispose()`。
* ECSList / ECSDictionary 的 Resize、Remove、Clear、Dispose 当前不设计为线程安全结构操作。
* managed 字段可以使用 EasyECS，也可以与 Unsafe Native Storage 共存，但 managed 字段自身仍由 GC 管理。
* 如果 Struct 完全没有 Native Storage 候选，即使开启 Allow Unsafe Code，也会优先选择 SafeSpan，缺少 Span 时选择 SafeRegistry。
* Hybrid Storage 会把含 managed 字段的 `[NotECS]` AoS Block 整体放入 Managed AoS，而不会把 managed reference 写入 Native Memory。

这些限制主要是为了避免给正常热路径增加额外运行时成本。

---

# ✅ 已验证环境

现有 Benchmark / Runtime 测试实际覆盖过：

```text
Unity 6000.3.21f1
Windows x64
Android ARM64
IL2CPP
```

Android 真机曾分别验证：

```text
Unsafe
SafeSpan
SafeRegistry
```

需要注意：

> **Managed Hybrid Unsafe 是上述真机验证之后新增的核心存储路径。**

因此发布包含 Hybrid Storage 的版本前，应使用最新 Generator 与 Runtime Unit Test 再验证一次 Hybrid 路径；没有实际验证的平台不在 README 中宣称完整兼容。

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
