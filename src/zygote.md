# app 进程怎么变成 `untrusted_app_34`（zygote 沙箱链路，源码 + 本机实测）

日期：2026-11-06　源码：`$HOME/work/aosp-src`（A15 快照）　设备：Android 17 / 无 root
 
## 一、源码里的链路（带行号）

1. **init 起 zygote**：`system_core/rootdir/init.zygote64.rc:1`
   `service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote`
   第 6 行 `socket zygote stream 660 root system`。此时进程是 **root + `zygote` 域**，不是 app。
2. **app_process 是入口**：`frameworks_base/cmds/app_process/app_main.cpp:79 AppRuntime::onStarted()`
   → 启动 `com.android.internal.os.ZygoteInit`，之后由 Java 侧 `Zygote.forkAndSpecialize()` 逐个 fork。
3. **每个 app 进程的降权顺序**（`frameworks_base/core/jni/com_android_internal_os_Zygote.cpp`）：

| 行号 | 动作 |
|---|---|
| 552 | `setgroups(...)` 设置附加组 |
| 2010 | `SetUpSeccompFilter(uid, is_child_zygote)` ← **先装 seccomp** |
| 2001 / 2015 | `setresgid` / `setresuid` ← 换成 app uid（如 10104） |
| 2133 | `selinux_android_setcontext(uid, is_system_server, se_info, nice_name)` ← **切 SELinux 域** |

`sandbox 三件套` = **uid + seccomp + SELinux 域**，一次 fork 里全部完成。

## 二、域是怎么选出来的（本机文件，一手）

`/system/etc/selinux/plat_seapp_contexts`（root 0644；**app 读不到**，`adb_shell` 能读）第 41 行：

```
user=_app minTargetSdkVersion=34 domain=untrusted_app_34 type=app_data_file levelFrom=all
```

- 我们的进程：`/proc/self/attr/current` = `u:r:untrusted_app_34:s0:c104,c256,c512,c768`
  → 命中 `minTargetSdkVersion=34`，且本 app 的 targetSdk 是 34。
- `type=app_data_file` 就是"应用数据文件"的类型 —— 与前面 execve 被拒用同一个类型名。
- `levelFrom=all` 生成了 `c104,c256,c512,c768` 这串 MLS 类别；`ls -Z` 里我们的文件带同样后缀，所以
  **同一个 app 的文件只有同一 app（同类别）能读**，这是应用隔离的另一半。

## 三、seccomp 实测（本机）

- `/proc/self/status`：`Seccomp: 2`、`Seccomp_filters: 1` → 过滤器已生效（mode 2 = FILTER）。
- 直接 exec 私有 ELF 的报错是 **EACCES（Permission denied）**，不是 seccomp 的 EPERM/ENOSYS
  → 再次证明拦路的是 SELinux 的 `entrypoint`，不是 seccomp。

## 四、上游源码与本机的漂移（值得记一笔）

- A15 源码 `SetUpSeccompFilter()`（`com_android_internal_os_Zygote.cpp:639`）按 uid 调
  `set_app_seccomp_filter()` / `set_system_seccomp_filter()`，历史上策略来自文本文件
  `/system/etc/seccomp_policy/app.arm64.policy`。
- 本机 Android 17：**该文件不存在**；`/system/etc/seccomp_policy/` 只剩 6 个（code_coverage、crash_dump、
  mediacodec、mediaextractor）。实际提供者是 **`/system/lib64/libseccomp_policy.so`**（36 KB，
  导出符号 `set_app_seccomp_filter`）。即策略从文本文件搬进了 so。

## 五、对"能不能在本机跑东西"的意义

1. 我们的 app 永远是 `untrusted_app_34` + `app_data_file` + 1 层 seccomp，改不了（无 root、无 BL）。
2. `execve` 的拒绝来自 SELinux entrypoint；`mmap` 加载 OK —— 所以 `/system/bin/linker64 <ET_DYN>`
   这条旁路在原理上是长期稳的，不是漏洞利用。
3. 需要 fork/exec 子进程的工具（git 的 `git-remote-https`、pip、多进程命令行）在 app 身份下**结构上不可行**，
   只能整条走 `adb_shell`（shell 域对 `shell_data_file` 有 entrypoint）。

## 复现命令

```sh
cat /proc/self/attr/current
grep -E '^Seccomp' /proc/self/status
adb_shell: grep -n 'untrusted_app' /system/etc/selinux/plat_seapp_contexts
adb_shell: ls -l /system/lib64/libseccomp_policy.so
```
