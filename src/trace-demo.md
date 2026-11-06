# 没有编译器，也能追一追 AMS 的调用链

> 设备：荣耀 Magic8 RSR（BKQ-AN20），Android 17，aarch64，无 root；源码在手机上（2.1 GB 快照）；
> 工具：自研 DSH 插件 `dsh-aosp@0.2.0`，内核只有两样东西 —— **ripgrep** 和 **花括号配对**。

## 为什么这么做

Android 的框架代码太大（`ActivityManagerService.java` 单文件 19 483 行），手机上也跑不了 IDE 的语言服务。
但读框架时真正费时的不是"读"，而是"定位"：这个方法被谁调、它又调了谁、链路上哪一步是入口。

于是用最笨也最稳的办法：**rg 找定义 → 花括号配对截函数体 → 从函数体里抽 `名字(`**，逐层展开。

```
aosp_trace {"symbol":"attachApplicationLocked","direction":"both","depth":2}
```

## 实测输出（真实内容，未做美化）

### 1. `attachApplicationLocked`（双向，深度 2，耗时 3.1 s）

```
L0 定义（2）
 ActivityManagerService.java:4506  private void attachApplicationLocked(@NonNull IApplicationThread thread,…
 ActiveServices.java:6867          boolean attachApplicationLocked(ProcessRecord proc, String processName)

L1（8 条，通用名未展开 3）
 ⇒ setBackgroundRestricted        ← ActiveServices.java:6867
 ⇒ appRestrictedAnyInBackground   ← ActiveServices.java:6867
 ⇐ attachApplication()            ← ActivityManagerService.java:4843
 ⇐ finishAttachApplicationInner() ← ActivityManagerService.java:4901
 ·（未展开：uptimeMillis, get, size）

L2（21 条，通用名未展开 4）
 ⇒ getAppStateTracker             ← ActiveServices.java:864
 ⇐ handleProcessStart()           ← ProcessList.java:2211
 ⇐ isProcStartValidLocked()       ← ProcessList.java:2869
```

**读出来的东西**：`attachApplicationLocked` 同名两处 —— 应用侧（AMS，收 `IApplicationThread`）和服务侧
（ActiveServices，收 `ProcessRecord`）；上行入口是 `attachApplication()`，下行有 `handleProcessStart()` 和
`isProcStartValidLocked()` 等。这条链就是"应用进程 attach 到 AMS"的主干。

### 2. `setBackgroundRestricted` 的调用者（会往上爬，791 ms）

```
L1 ⇐ updateAppRestrictedAnyInBackgroundLocked()  ← ActiveServices.java:878
   ⇐ createAppZygoteForProcessIfNeeded()         ← ProcessList.java:2543
L2 ⇐ stopInBackgroundLocked()                    ← ActiveServices.java:1709
   ⇐ appRestrictedAnyInBackground()              ← ActiveServices.java:1019
```

关键点：工具不是只给出"某行调用了它"，而是**定位到包含这一行的外层函数**再继续往上追——否则 callers
方向第二层就没东西可追了（第一版就栽在这）。

### 3. `handleBindApplication`（应用侧主入口，向下看）

同样一次调用内可以换符号继续，无需重新建索引：定义在 `ActivityThread.java`，展开后能看到
`Instrumentation`/`Application` 相关的初始化调用。

## 数字

| 项目 | 值 |
|---|---|
| 单次追踪耗时 | **281 ms**（单层）– **3.1 s**（双向 2 层、21 条 L2） |
| 单层节点上限 | 默认 5，可调到 10 |
| 通用名过滤 | 出现在 **> 8 个文件**的名字不继续展开（否则 `getPid`/`split`/`parseInt` 会刷屏） |
| 目录排除 | 默认排除 `tests`/`test`/`androidTest` |

## 局限（必须一起说）

- 重载只按**名字**匹配，`attachApplicationLocked(ProcessRecord…)` 与 `attachApplicationLocked(@NonNull IApplicationThread…)`
  会被当成同一个符号；
- **反射、宏、模板、字符串里的调用**看不到；
- 花括号配对是纯文本，**字符串/注释里的 `{`** 可能让它提前收尾；
- 跨模块同名函数会被混在一起。

所以它回答的是"这条链上大概都有谁"，不是精确调用图。真要精确，得出 Android Studio 或 Sourcegraph。

## 一句话

在一台不能编译、不能 run IDE 的手机上，**rg + 花括号配对 + 一层层收窄**已经足够把框架主干摸清；
成本是一次 300 ms 到 3 秒，换来的是不用再靠记忆猜"这个方法在哪被调"。

---

*工具：`dsh-aosp@0.2.0`（9 个工具：list/preset/mirror/fetch/search/grep/trace/read/update）；
源码 `Verchiel666/dsh-aosp`；源码树快照来自 GitHub `aosp-mirror`（≈ Android 15/16）。*
