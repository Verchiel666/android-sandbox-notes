# 本机（荣耀 Magic8 RSR / Android 17 / 无 root）跑 AOSP 下载-编译-刷机 可行性实测

日期：2026-11-06　机型：BKQ-AN20（荣耀Magic8 RSR 保时捷设计），SM8850，Android 17 / SDK 37，aarch64
环境：DSH app（uid 10104，untrusted_app_34）+ Shizuku shell（uid 2000）

## 结论
文章《一文搞掂Android AOSP的下载、编译、刷机》的三步，本机**三步都不可行**，但与 root 无关：
不可行的是①x86_64 Linux 编译主机 ②解锁 BL（刷机通道）③荣耀无 AOSP device tree / vendor blobs。
唯一可行的是"读源码"——但要换路径（GitHub codeload 的 tar 快照），不能用 repo/git 全量同步。

## 实测数据

### 1) USTC 镜像 git clone：失败（卡死）
- 位置：https://mirrors.ustc.edu.cn/aosp/platform/{build,packages/apps/Settings,...}
- 现象：ref advertisement 能拿到（manifest.git/info/refs 750 KB / 0.6 s），但 POST /git-upload-pack 之后
  进度恒停在 **119 KB**，1–4 分钟无增长，HTTPS / HTTP / HTTP1.1 / 去掉 Expect 四种组合全都一样。
- 判定：不是 TLS，不是快照深度，是链路（或镜像侧）对 fetch 流量的阻塞。

### 2) GitHub codeload tar 快照：可用（推荐）
- 组织：aosp-mirror（100 个仓库，覆盖 frameworks/base、packages/apps/*、system/*、build/* 等主要项目）
- 实测：
  - platform_packages_apps_Settings（main）：46.2 MB / 7.0 s（≈6.7 MB/s）→ 解压 167 MB / 8087 个文件
  - platform_system_core（main）：1.9 MB / 2.8 s → 解压 18 MB / 1810 个文件
  - 校验：src/com/android/settings/homepage/SettingsHomepageActivity.java 862 行，可 grep
- 版本可选：tags 里有 **android-12.1.0_r1**（文章用的版本）、android-14.0.0_r1、android-15.0.0_r1，
  refs/tags 的 tarball 返回 200 application/x-gzip。
- 脚本：$HOME/work/aosp-get.mjs
  `node aosp-get.mjs packages/apps/Settings android-12.1.0_r1 ./a12`
  映射规则：<路径>/<a> → aosp-mirror/platform_<a>_<b>

### 3) git 本身能在本机跑（但只能以 shell 身份）
- app 身份：git clone 立即 `fatal: cannot exec 'remote-https': Permission denied`（SELinux 禁止 app 私有目录 execve）
- shell 身份：把 termux prefix 复制到 /data/local/tmp/tp + 包装 /data/local/tmp/gitx.sh（设 GIT_EXEC_PATH/GIT_SSL_CAINFO）
  后 `git version 2.56.0` 正常、helper 能 exec；失败原因是上面的镜像卡死，不是 git 本身。
- 注意：CURL_CA_BUNDLE 不生效，必须用 GIT_SSL_CAINFO 指向 prefix/etc/tls/cert.pem。

### 4) 编译 / 刷机：硬墙
- 官方要求 64 位 x86 主机 + 400 GB 空闲 + 64 GB 内存；本机 aarch64、无 apt、无 python/gcc/clang/java，
  AOSP host 工具全是 prebuilts/build-tools/linux-x86 的 x86_64 ELF，跑不了。
- 内存：23.7 GB（可用 6.5 GB），已知 26 GiB + 4 GiB swap 的机器 -j2 仍被 oomd 杀。
- BL：ro.boot.flash.locked=1、verifiedbootstate=green；荣耀官方（2026-04-25）明确不提供解锁码。
- AOSP 只带 Pixel / goldfish / cuttlefish 设备配置，荣耀无 device tree / vendor blobs。

## 复现命令
```
node $HOME/work/aosp-get.mjs packages/apps/Settings    # 约 46 MB / 7 s
grep -rn "class SettingsHomepageActivity" aosp_*       # 解压后
```
