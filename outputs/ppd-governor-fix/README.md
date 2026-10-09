# PPD Intel P-State 修复产物索引

此目录不再保存 `.deb` 二进制文件。修复包和用于回滚的 Ubuntu 原版包均从 GitHub Release 下载：

- [修复版 `power-profiles-daemon_0.30-2xing1_amd64.deb`](https://github.com/xingwangzhe/ubuntu-intel-pstate-performance-governor/releases/download/v0.30-2xing1/power-profiles-daemon_0.30-2xing1_amd64.deb)
- [原版 `power-profiles-daemon_0.30-2_amd64.deb`（回滚用）](https://github.com/xingwangzhe/ubuntu-intel-pstate-performance-governor/releases/download/v0.30-2xing1/power-profiles-daemon_0.30-2_amd64.deb)

源码仓库中的对应材料：

- C 实现：[`src/ppd-driver-intel-pstate.c`](../../src/ppd-driver-intel-pstate.c)
- 完整源码补丁：[`intel-pstate-performance-governor.patch`](../../intel-pstate-performance-governor.patch)
- 上游源码和 Debian packaging：[`source/`](../../source/)
- 构建、安装、回滚及许可说明：[`README.md`](../../README.md)

本索引不包含聊天记录、用户目录路径、个人邮箱、构建日志或二进制文件。
