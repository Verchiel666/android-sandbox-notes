# 为什么 `linker64 <file>` 能跑、直接 exec 不能（源码 + 本机实测）

日期：2026-11-06　源码：`$HOME/work/aosp-src/bionic`（aosp-mirror，main）

## 一句话
`execve` 被拒是 **SELinux 的 `file:entrypoint`**（目标类型 `app_data_file` 不给 app 当入口点）；
能跑是因为 `/system/bin/linker64` 的类型是专门的 **`system_linker_exec`**，它作为"解释器"去
**mmap + mprotect(PROT_EXEC)** 我们的文件（这属于 `file:execute`，app 对自己的 data 目录是允许的）。

## 本机实测（有编号，可复现）
| 观察 | 结果 |
|---|---|
| 私有文件模式 | `-rwxr-xr-x`（0755），uid 就是我们自己 |
| `/data` 挂载选项 | `f2fs rw,...,seclabel,nosuid,nodev` —— **没有 noexec**，排除 mount 因素 |
| 直接跑 `.../usr/bin/jq` | `Permission denied`（EACCES） |
| `linker64 .../jq`（不带 LD_LIBRARY_PATH） | `CANNOT LINK EXECUTABLE ...: library "libjq.so" not found` ⇒ 已经进入 bionic 加载阶段，说明 mmap 成功 |
| `LD_LIBRARY_PATH=.../lib linker64 .../jq` | **jq-1.8.2**（正常输出） |
| `linker64 $HOME/../bin/rg` | `has unexpected e_type: 2`（rg 是 ET_EXEC 全静态） |

SELinux 类型（`ls -Z` / `getenforce` = Enforcing）：
- `u:object_r:system_linker_exec:s0 /system/bin/linker64`
- `u:object_r:shell_exec:s0 /system/bin/sh`
- `u:object_r:app_data_file:s0:c104,c256,c512,c768` ← 我们的私有目录、tbins prefix、APK 解出来的 `libdshnode.so`

## bionic 源码依据
1. **只收 ET_DYN**：`linker/linker_phdr.cpp:292-294`
   ```cpp
   if (header_.e_type != ET_DYN) {
     DL_ERR("\"%s\" has unexpected e_type: %d", name_.c_str(), header_.e_type);
     return false;
   }
   ```
   ET_EXEC = 2 ⇒ 静态/PIE 之外的二进制一律被拒，和 `librg.so` 的实测报错逐字对应。
2. **支持"直接当加载器跑"**：`linker/linker_main.cpp:790-823`
   ```cpp
   // When the linker is run directly rather than acting as PT_INTERP, parse
   // arguments and determine the executable to load. ...
   exe_to_load = args.argv[1];
   ```
   官方 usage 字符串也写着：*"This program can also be run directly to load and run a dynamic executable."*
   所以 `/system/bin/linker64 <elf>` 是**受支持用法**，不是野路子。
3. 加载过程走 `mmap` + `mprotect(PROT_READ|PROT_WRITE)`（`linker_phdr.cpp:597-705`），
   与我们观察到的"同目录 .so 能加载、execve 不能"一致：前者是 `execute`，后者要 `entrypoint`。

## 对我们的影响
- 私有目录里的工具**必须是 ET_DYN（PIE）+ bionic 动态链接**；静态/ET_EXEC 无解（除非走 shell 身份）。
- 依赖库必须让 linker 找得到：`LD_LIBRARY_PATH=<prefix>/lib`（`dsh-termux`/`tb` 包装器自动设），
  或编译时带 `RPATH=$ORIGIN`（`libdshnode.so` 就是这么找到 `libnode.so` 的）。
- 需要 exec 子进程的工具（git 的 `git-remote-https`、python 的 pip 等）**根本走不通**：
  子进程 execve 仍然撞 `app_data_file` 的 entrypoint 限制 —— 这正是 git 只能以 shell 身份跑的原因。
