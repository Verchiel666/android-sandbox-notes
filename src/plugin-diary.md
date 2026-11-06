# 在一台不 root 的 Android 手机上给 DSH 写插件：3 次真实失败与修复记录

> 目标：让 DSH 能直接"拉一个 AOSP 模块的源码并在本机搜代码"。
> 约束：没有 root、没有 git 二进制、`github.com` 不通、只有 node（v24）和一个能发 REST 的 GitHub token。
> 结果：`dsh-aosp`，5 个版本，最后 `npm i dsh-aosp@0.1.5`。

这篇文章不讲"怎么写插件"（那是文档的事），只讲**这次真实遇到的坑**——每个都有当时的输出或数字。

---

## 坑 1｜npm 新包首发是"暂存"的，同版本重发会被**静默**吞掉

第一次 `npm publish` 输出很漂亮：

```
npm notice name: dsh-aosp
npm notice version: 0.1.0
+ dsh-aosp@0.1.0
```

但 registry 上查到的却是：

```
versions: 0.0.0-stage
description: Temporary package placeholder for staged publishing
```

`0.0.0-stage` 是 npm 新包首发时先出现的占位符，真实版本 1–2 分钟后才可见。**第一次翻车就在这里**：我加完第 5 个工具（`aosp_grep`）后把版本号改成 0.1.1 重新发——发布"成功"，可下载 tarball 加载后只有 4 个工具。

原因是 0.1.1 这个版本号在之前的试探里已经进过暂存队列，重发得到的是：

```
npm error 409 Conflict - PUT https://registry.npmjs.org/dsh-aosp
npm error Cannot publish over previously staged version "0.1.1"
```

而这条错误被我的 `tail` 过滤掉了，于是 registry 最后上线的仍是旧内容的 0.1.1。

**修法**（已固化成发布后检查表）：

1. 发布后**不要相信输出**，下载 tarball、解包、真实 `import` 一次、把工具列表打出来；
2. 内容不对就**换版本号重发**（0.1.2 才真正带上 `aosp_grep`）；
3. 查 `<pkg>`、`dist-tags.latest`、`versions` 三处，以 registry 为准。

---

## 坑 2｜codeload 的 tar 包目录名里**没有** commit sha

想要"增量更新"，就得知道本地这份源码对应哪个 commit。第一版实现想省一次 API 调用：从 tar 包顶层目录名取 sha。

```
首行 = "platform_external_zlib-main/"   ← 分支 ref 时是分支名，不是 sha
tarballSha = null
```

只有按 sha 下载时 GitHub 才会把目录名写成 `repo-<sha>/`。

**修法**：反过来做——先用 API 查 ref 的 sha，**再按 sha 下载**：

```js
const { sha } = await remoteHead(repo, ref);      // /commits/refs/heads/main
await downloadTarball(project, sha, tgz);          // /tar.gz/<sha>
```

好处是包内容与记录的 sha 严格一致，增量比对不会出现"本地比记录新/旧"的错位。

---

## 坑 3｜jsDelivr 的路径必须带组织名，否则 400

增量更新只下载改动文件，靠 jsDelivr 镜像取单文件（`raw.githubusercontent.com` 在本机不通）。第一版：

```
取文件失败 400：Android.bp
```

响应正文写着：`Invalid URL. The URL structure is /gh/user/repo@version/file.js` —— 少了组织名。

**修法**：

```
https://gcore.jsdelivr.net/gh/aosp-mirror/platform_external_zlib@<sha>/Android.bp
```

实测同样按 commit 取文件：`200 3392B`。

---

## 坑 4｜解包 1.8 GB 要 551 秒，600 秒的超时会在最后一刻杀掉它

第一次重拉 `frameworks/base` 得到：

```
❌ frameworks_base: 解包失败（tar null）：
```

`tar null` = 被信号终止 —— 我给 tar 设的 600 秒超时到了。当时目录里只有 158 MB / 3146 个文件。

第二天（同一天晚些）用 30 分钟超时重跑：

```
[0.8s]   下载 frameworks/base@main #1cdfff5 …
[70.3s]  解包 1148.5 MB …
[621.2s] 解包完成 550.7s
✅ 1713.3 MB 下载 / 69.5s，解包 551.1s，56733 文件
```

**反直觉的数据**：解包耗时不是按字节线性，而是按**文件数**：

| 模块 | 压缩包 | 解压后 | 文件数 | 解包耗时 |
|---|---|---|---|---|
| packages/apps/Settings | 146 MB | 167 MB | 8 087 | **2.1 s** |
| frameworks/base | 1713 MB | 1.8 GB | 56 733 | **551 s** |

即 ≈100 文件/秒。600 秒只差 50 秒，随便跑点别的就超时。

**修法**：tar 超时 30 分钟；下载失败自动重试 3 次；日志里打印真实解包耗时，下次不必再猜。

---

## 坑 5｜列表工具每次重扫 56k 文件，卡到 5 分钟

`aosp_list` 原本每次都用 `readdir` 递归统计每个模块的文件数/体积。四个模块、约 7 万个文件时，实测**卡住超过 5 分钟**（我的工具调用都超时了）。

**修法**：把计数写进 `.dsh-aosp.json`，列表直接读缓存（**14–17 ms**），要重扫时显式传 `refresh:true`。缓存写入时机是"下载/更新完成时"。

```
listModules（缓存）耗时 14 ms
 frameworks_base    1713 MB 56733 文件 #1cdfff5
 settings-src        146 MB  8087 文件 #7c59825
 bionic               54 MB  3451 文件 #731631f
 system_core          12 MB  1810 文件 #a3b721a
```

---

## 工程侧：三层验证，缺一层就会漏

这台机器上没有"跑一下试试"的便利，所以每个版本都过三关：

| 层 | 内容 | 最后一次通过数 |
|---|---|---|
| 离线单测 | 项目名映射、路径穿越防护、行号分页、模块聚合、缓存命中、插件注册 | **20 项** |
| 联网测试 | 真下载、真按 sha 记录、真增量（把 meta 退回旧 sha 再更新） | **24 项** |
| 引擎冒烟 | 引擎里真的 cordis 4.0.5：`inject`、注册、`execute`、审批钩子、`dispose` 清理 | **12 项** |

审批钩子值得一提：`aosp_fetch` / `aosp_update` 会写盘，于是把 `tools/pre-execute` 的 `allow` 升级成 `ask`，
但**上游的 deny 不许被改写**（这条也有测试）。

---

## 版本流水

| 版本 | 变化 | 教训来源 |
|---|---|---|
| 0.1.0 | 4 个工具，首发 | staged 占位 |
| 0.1.1 | 加 `aosp_grep`——**实际没进包** | 坑 1 |
| 0.1.2 | 重发，5 个工具 | 发布后验证 |
| 0.1.3 | `aosp_update` 增量更新 | 坑 2、坑 3 |
| 0.1.4 | 下载重试 3 次 / tar 超时 30 分钟 | 坑 4 |
| 0.1.5 | `aosp_list` 用缓存 + `refresh` | 坑 5 |

---

## 可以带走的三条

1. **发布后必须验证产物本身**：拉 tarball、真加载、打工具清单。这比看 CI 绿更可靠——尤其当输出被管道过滤时。
2. **超时要按最坏情况设**：这台手机上小文件解包只有 ≈100 文件/秒，1.8 GB 的源码树要 9 分钟。
3. **"慢"也是一种 bug**：列表工具重扫 7 万个文件要几分钟，改成缓存后 14 ms——用户感知的差别比功能增减更大。

---

*设备：荣耀 Magic8 RSR（BKQ-AN20），Android 17 / SDK 37，aarch64，无 root；插件 `dsh-aosp@0.1.5`，源码在 `Verchiel666/dsh-aosp`，站点 `verchiel666.github.io/android-sandbox-notes`。*
