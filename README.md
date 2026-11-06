# 无 root 手机上的 Android 沙箱实测

用 AOSP 源码 + 真机实测，讲清一台不 root 的 Android 手机上，一个普通 App 能跑什么、不能跑什么。

三层的答案：
1. **SELinux `file:entrypoint`** —— 为什么 `execve` 被拒，而 `/system/bin/linker64 <ET_DYN>` 可以（bionic 源码依据）。
2. **zygote 一次 fork 定死的身份** —— `untrusted_app_34` + `app_data_file` + MLS 类别 + seccomp 怎么落到进程上。
3. **init 发出的 socket 权限模型** —— 660 root:system + `zygote_socket` 标签 + `SO_PASSCRED` peer cred。

站点：https://verchiel666.github.io/android-sandbox-notes/

- `index.html` 单页版（移动端优先，支持深色模式）
- `plugin-diary.html` 第二篇：[给 DSH 写插件的 3 次真实翻车](https://verchiel666.github.io/android-sandbox-notes/plugin-diary.html)
- `src/*.md` 原始 Markdown（含源码行号与复现命令）

设备：荣耀 Magic8 RSR（BKQ-AN20），Android 17 / SDK 37，aarch64，无 root，Shizuku shell。
源码快照：GitHub `aosp-mirror`（≈ Android 15/16），与本机 Android 17 的漂移在文中标注。
