# 无 root 手机上，一个 App 到底能跑什么、不能跑什么

> 设备：荣耀 Magic8 RSR 保时捷设计（BKQ-AN20，SM8850），Android 17 / SDK 37，kernel 6.12.58，aarch64
> 身份：普通 App，uid 10104，targetSdk 34；辅助身份：Shizuku 提供的 shell（uid 2000）
> 方法：AOSP 源码（frameworks/base、system/core、bionic 快照）+ 本机实测，两边对齐

三层的沙箱把答案写死了：**SELinux entrypoint（能不能执行）、zygote 一次 fork 定死的身份（你是谁）、
init 发出去的 socket 权限模型（能跟谁说话）**。

---

## 第一层｜execve 被拒，但 linker64 能跑

### 排除法：不是文件属性，也不是挂载
| 检查 | 结果 |
|---|---|
| 私有 ELF 权限 | `-rwxr-xr-x`，属主就是自己 |
| `/data` 挂载 | `f2fs rw,seclabel,nosuid,nodev` —— 没有 `noexec` |
| 直接执行 | `Permission denied`（EACCES） |

### 真正的原因：SELinux `file:entrypoint`

| 文件 | SELinux 类型 |
|---|---|
| `/system/bin/linker64` | `system_linker_exec`（专门的类型） |
| `/system/bin/sh` | `shell_exec` |
| App 私有目录、`libdshnode.so` | `app_data_file` |

App 对前两种类型有 `entrypoint` 权限，对 `app_data_file` 没有 → execve 直接被内核拒绝。

### 为什么 `linker64 <file>` 可以：加载 ≠ 执行
bionic 的 linker 本来就是"解释器"角色，它支持的用法里明确写了直接跑：

- `bionic/linker/linker_main.cpp:790-823`
  ```cpp
  // When the linker is run directly rather than acting as PT_INTERP, parse
  // arguments and determine the executable to load. ...
  exe_to_load = args.argv[1];
  ```
  官方 usage：*"This program can also be run directly to load and run a dynamic executable."*
- 它用 `mmap` + `mprotect(PROT_EXEC)` 把我们的文件映射进来（`linker_phdr.cpp:597-705`），走的是
  `file:execute`——和 App 加载自己的 `.so` 是同一条权限路径，不触发 entrypoint 检查。

### 代价：只能跑 ET_DYN（PIE）
`bionic/linker/linker_phdr.cpp:292-294`：
```cpp
if (header_.e_type != ET_DYN) {
  DL_ERR("\"%s\" has unexpected e_type: %d", ...);
```
实测 `linker64 librg.so`（ET_EXEC 全静态）→ `has unexpected e_type: 2`，与源码逐字对上。

### 依赖库要自己给路径
```
linker64 jq            → CANNOT LINK EXECUTABLE: library "libjq.so" not found
LD_LIBRARY_PATH=... bin → jq-1.8.2
```
（同理，`libdshnode.so` 靠编译期 `RPATH=$ORIGIN` 找到 82 MB 的 `libnode.so`。）

**推论**：任何需要 fork/exec 子进程的工具（git 的 `git-remote-https`、pip、多进程 CLI）在 App 身份下
**结构性不可行**，因为子进程的 execve 会再撞一次 `app_data_file` 的 entrypoint 限制。

---

## 第二层｜App 的沙箱身份是 fork 时一次定死的

### 源码链路
1. `system/core/rootdir/init.zygote64.rc:1`：
   `service zygote /system/bin/app_process64 --zygote ...`，`socket zygote stream 660 root system`
   —— 此刻是 **root + `zygote` 域**（本机实测：`zygote64` PID 4337，user=root，label=`u:r:zygote:s0`）。
2. `frameworks/base/cmds/app_process/app_main.cpp:79` → `ZygoteInit` → 每个 App 由 `forkAndSpecialize()` 生出。
3. `frameworks/base/core/jni/com_android_internal_os_Zygote.cpp` 里依次做完：

| 行号 | 动作 | 意义 |
|---|---|---|
| 552 | `setgroups(...)` | 附加组 |
| 2010 | `SetUpSeccompFilter(uid, …)` | 装 seccomp 过滤器 |
| 2001 / 2015 | `setresgid` / `setresuid` | 换成 App 的 uid |
| 2133 | `selinux_android_setcontext(uid, is_system_server, se_info, name)` | 切 SELinux 域 |

### 域从哪来（本机文件，一手）
`/system/etc/selinux/plat_seapp_contexts:41`（App 读不到，`adb_shell` 能读）：
```
user=_app minTargetSdkVersion=34 domain=untrusted_app_34 type=app_data_file levelFrom=all
```
我们的进程 `/proc/self/attr/current` = `u:r:untrusted_app_34:s0:c104,c256,c512,c768`：
- `minTargetSdkVersion=34` 命中（本 App targetSdk 34）
- `type=app_data_file` —— 就是第一层 execve 被拒用的类型
- `levelFrom=all` 生成 `c104,c256,c512,c768` 这串 MLS 类别：同 App 的文件只有同类别能读，是隔离的另一半

### seccomp 在，但不是拦路的那位
`/proc/self/status`：`Seccomp: 2`、`Seccomp_filters: 1`（过滤器已生效）。
但 exec 返回的是 **EACCES（SELinux）**，不是 seccomp 的 EPERM —— 两者可以据此区分。

### 一处处"文档对不上本机"
A15 源码里策略来自文本文件 `/system/etc/seccomp_policy/app.arm64.policy`；
**Android 17 上该文件不存在**，实际提供者是 `/system/lib64/libseccomp_policy.so`（36 KB，
导出符号 `set_app_seccomp_filter`）——策略从文件搬进了 so。

---

## 第三层｜socket 权限模型：谁能和谁说话

### init 发 socket 时定了三件事
`system/core/init/util.cpp:95 CreateSocket(name, type, passcred, listen, perm, uid, gid, socketcon)`：
1. `setsockcreatecon(socketcon)` —— 创建时就指定 socket 的 SELinux 标签；
2. `SelabelLookupFileContext` + `setfscreatecon` —— 按 `file_contexts` 给 socket 节点打标签；
3. `SO_PASSCRED` —— 接收端可拿到对端 uid/gid/pid（`SCM_CREDENTIALS`），用于身份核对。
调用方 `init/service_utils.cpp:199` 把 `.rc` 里的 `perm/uid/gid` 传进去。

### 本机实测
- `.rc`：`socket zygote stream 660 root system`（另有 `zygote_next seqpacket 660 root system`）
- `/system/etc/selinux/plat_file_contexts`：`/dev/socket/zygote u:object_r:zygote_socket:s0`
- `/proc/net/unix` 里确有 `/dev/socket/zygote`
- **代价是连 stat 都不给**：`ls -l /dev/socket` 列出 59 个条目，socket 文件那一批全是
  `s?????????`（stat 被拒）；App 或 shell 直接 `ls -l /dev/socket/zygote` → `Permission denied`

所以一个普通 App 想"假装"和 zygote/servicemanager 通信，会在**权限位、SELinux 标签、peer cred**
三关里任一关被挡下；这也是第三方 App 无法劫持系统 socket 的底层原因。

---

## 结论：能做 / 不能做（对无 root 的 App 成立）

| 能力 | 结果 | 依据 |
|---|---|---|
| 加载并运行**自己目录里的 ET_DYN 动态二进制** | ✅ 可以（`/system/bin/linker64 <路径>`） | SELinux 只拦 execve，不拦 mmap |
| 跑静态 / ET_EXEC 二进制 | ❌ | bionic 只收 ET_DYN |
| fork/exec 子进程（git 的 https helper、pip…） | ❌ | 子进程 execve 撞 entrypoint |
| 读自己 App 的数据文件 | ✅ | 同 MLS 类别 `c104,…` |
| 读别的 App 数据、系统 sealed 文件（seapp_contexts 等） | ❌ | 类型 + 类别 |
| 连系统 socket（zygote、property_service…） | ❌ | 660 root:system + `zygote_socket` 标签 |
| 借 shell（Shizuku）跑任意 aarch64 原生程序 | ✅ | shell 域对 `shell_data_file` 有 entrypoint |

**一句话**：这台手机上"能不能跑某个程序"不是编译器问题、不是权限位问题，而是
**它落在哪个 SELinux 类型 + 哪些 MLS 类别 + 由谁 fork 出来**。

---

## 复现命令

```sh
# 1. execve vs linker64
ls -l <你的私有二进制>; <二进制>            # Permission denied (EACCES)
/system/bin/linker64 <你的私有二进制>        # 需要 LD_LIBRARY_PATH

# 2. 自己的沙箱身份
cat /proc/self/attr/current
grep -E '^Seccomp' /proc/self/status
adb_shell: grep -n untrusted_app /system/etc/selinux/plat_seapp_contexts
adb_shell: ls -l /system/lib64/libseccomp_policy.so

# 3. socket 模型
adb_shell: ls -lZ /dev/socket | head
adb_shell: grep zygote /proc/net/unix
adb_shell: grep "socket/zygote" /system/etc/selinux/plat_file_contexts
```

## 源码位置（本机快照）
`$HOME/work/aosp-src`：`bionic/`、`system_core/`、`frameworks_base/`、`settings-src/`
（来源：GitHub `aosp-mirror` tar 快照；镜像最后 push 2025-11-12，≈ Android 15/16，
本机是 Android 17，个别实现已有漂移，文中已标注。）
