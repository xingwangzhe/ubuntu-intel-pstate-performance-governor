[中文](#中文) | [English](#english)

# 中文

**这次问题的完整排查经过、补丁思路和实机验证记录，见我的博客文章：[Ubuntu Intel 性能模式问题排查：我修复 i7-1260P 的 PPD Governor 切换](https://xingwangzhe.fun/posts/ubuntu-intel-pstate-performance-governor/)。**

## 项目说明

这是基于 Ubuntu `power-profiles-daemon 0.30-2` 重打包的本地实验版本，针对一台笔记本上“性能模式已切换，但 CPUFreq governor 仍为 `powersave`”的问题调整 Intel P-State 的 profile 应用逻辑。

## 已验证环境

- CPU：**Intel Core i7-1260P**（第 12 代，16 个逻辑 CPU）
- 系统：**Ubuntu 26.04.1 LTS（Resolute Raccoon）**
- 内核：**7.0.0-38-generic**
- 调频驱动：`intel_pstate`，active 模式
- 固件平台 profile：不可用（PPD 显示 `PlatformDriver: placeholder`）
- 修复包：`power-profiles-daemon 0.30-2xing1`，`amd64`

这是针对上述单机配置验证的非官方实验包，不是 Intel 某一批次或步进的故障结论，也不代表其他处理器、固件、内核或 PPD 版本。未经验证的组合不保证适用。

## 补丁行为

- 切到 `performance` 时，将每个 EPP policy 的 `scaling_governor` 设为 `performance`，并跳过 EPP 写入。
- 切到 `balanced` 或 `power-saver` 时，先恢复 `powersave`，再写入 PPD 按 profile 和电源状态选择的 EPP。
- 不增加额外的 systemd 服务、D-Bus 监听器或内核启动参数，也不调整散热配置。

在 active `intel_pstate` 的性能算法管理 EPP 时，写入非性能 EPP 可能被内核以 `EBUSY` 拒绝。补丁通过调整同一个 PPD profile 激活流程里的写入顺序来避开这一冲突。

## 关键 C 源码与上游改造

关键实现直接放在 [`src/ppd-driver-intel-pstate.c`](src/ppd-driver-intel-pstate.c)，这是基于 PPD 0.30-2 修改后的完整 Intel P-State 驱动 C 文件，保留了上游版权与 GPL-3 头部。便于应用到干净上游源码的完整改动（包括对应集成测试更新）见 [`intel-pstate-performance-governor.patch`](intel-pstate-performance-governor.patch)。`source/` 中同时提供生成修复包所基于的 Ubuntu 0.30-2 上游 tarball、Debian packaging tarball 和 `.dsc` 源码包描述文件。

这项修复需要**改造并重新构建 power-profiles-daemon 上游源码**：补丁修改 `src/ppd-driver-intel-pstate.c` 中 governor/EPP 应用顺序，也更新 `tests/integration_test.py` 对 performance 和回切行为的断言。它不是可单独安装的脚本或 sysfs 配置。可按以下步骤展开并应用补丁：

```sh
dpkg-source -x source/power-profiles-daemon_0.30-2.dsc
cd power-profiles-daemon-0.30
patch -p1 < ../intel-pstate-performance-governor.patch
dch --local xing1 "Use the intel_pstate performance governor for the performance profile"
dpkg-buildpackage -b -uc -us
```

源码包描述文件 `.dsc` 带有上游签名。用 `dpkg-source` 展开时，应确认签名由本机配置的可信 Debian/Ubuntu 密钥验证；如果工具报告缺少公钥或签名无法验证，就不能视为通过验证。`sha256sum -c SHA256SUMS` 可检查文件是否与仓库清单一致，但清单本身未签名，不能单独证明来源。然后按常规 Debian packaging 流程安装构建出的 `0.30-2xing1` 包，并按上面的验证步骤检查。该命令记录源码改造路径；不同构建环境可能需要先安装 PPD 的 build dependencies。

## 构建与验证记录

- 源码基线：Ubuntu `power-profiles-daemon 0.30-2`。
- Debian 包版本：`0.30-2xing1`。
- 本地记录的 PPD 测试套件：**127 项通过，0 项失败**。
- 在上述机器上实际完成 `balanced → performance → balanced` 往返：performance 档 16 个 governor 均为 `performance`；回到接 AC 的 balanced 档后，16 个 governor 均为 `powersave`，EPP 均为 `balance_performance`。检查时段内的 PPD journal 未发现匹配 `busy`、`failed`、`error` 或 `warning` 的记录。
- 没有进行控制变量跑分，不声称有特定吞吐量、频率或性能提升。

## 安装

从 [GitHub Release `v0.30-2xing1`](https://github.com/xingwangzhe/ubuntu-intel-pstate-performance-governor/releases/tag/v0.30-2xing1) 下载修复包。Release 同时提供原版 Ubuntu `0.30-2` 包用于回滚。安装会替换系统 PPD 包并改变系统电源策略；安装前请检查包信息，并确认系统依赖符合包元数据要求。

```sh
sha256sum -c SHA256SUMS
pkexec dpkg -i ./power-profiles-daemon_0.30-2xing1_amd64.deb
```

安装后检查 profile 和所有 policy：

```sh
powerprofilesctl set performance
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c

powerprofilesctl set balanced
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c
for f in /sys/devices/system/cpu/cpufreq/policy*/energy_performance_preference; do cat "$f"; done | sort | uniq -c
journalctl -u power-profiles-daemon --since '10 minutes ago' --no-pager | grep -Ei 'busy|failed|error|warning'
```

在已验证的 AC 场景中，预期 performance 档有 16 个 `performance` governor；balanced 档有 16 个 `powersave` governor 和 16 个 `balance_performance` EPP。最后一条命令无输出，只表示该时间范围内没有匹配这些词的日志。

## 回滚

```sh
pkexec dpkg -i ./power-profiles-daemon_0.30-2_amd64.deb
powerprofilesctl set balanced
```

## 限制与风险

`performance` governor 选择驱动的性能算法，不会把 CPU 固定在标称最高频率，也不能绕过固件功耗限制、温度限制或热节流。这个包修改特权系统电源策略，且不通过 Ubuntu 官方仓库分发。请先审阅补丁与包元数据，并保留原版包或其他恢复方式。

## 源码与许可

PPD 主程序与这里的 C 文件依 PPD 上游声明采用 GPL-3；完整对应的上游源码、Debian packaging 与本地改动一并提供，便于检查和重建二进制包。补丁改动到的 `tests/integration_test.py` 在上游 `debian/copyright` 中标注为 GPL-2-or-later。我们保留原作者版权声明，只在本地补丁和新增内容范围内署名，不把上游代码或其他人的版权据为己有。许可文本见 [`LICENSES/GPL-3.0.txt`](LICENSES/GPL-3.0.txt)、[`LICENSES/GPL-2.0.txt`](LICENSES/GPL-2.0.txt) 和 [`LICENSES/GFDL-1.3.txt`](LICENSES/GFDL-1.3.txt)；详细文件归属及适用版本以随附源码包中的 `debian/copyright` 为准。

---

# English

**For the full investigation, patch rationale, and on-device verification, see my blog post: [Ubuntu Intel Performance Mode Issue: Fixing PPD Governor Switching on an i7-1260P](https://xingwangzhe.fun/posts/ubuntu-intel-pstate-performance-governor/).**

## Overview

This is an unofficial local rebuild of Ubuntu's `power-profiles-daemon 0.30-2`. It changes the Intel P-State profile application path for one laptop where selecting the performance profile changed EPP but left CPUFreq policies on the `powersave` governor.

## Verified environment

- CPU: **Intel Core i7-1260P** (12th Gen, 16 logical CPUs)
- OS: **Ubuntu 26.04.1 LTS (Resolute Raccoon)**
- Kernel: **7.0.0-38-generic**
- Scaling driver: `intel_pstate`, active mode
- Firmware platform profile: unavailable (`PlatformDriver: placeholder` in PPD)
- Patched package: `power-profiles-daemon 0.30-2xing1`, `amd64`

This experimental package was verified on that specific machine configuration. It is not evidence of a defect affecting an Intel batch or stepping, and it is not an official Ubuntu package or a general performance guarantee. Other CPUs, firmware, kernels, and PPD versions have not been validated.

## Patch behavior

- When switching to `performance`, set `scaling_governor` to `performance` for each EPP policy and skip the EPP write.
- When switching to `balanced` or `power-saver`, restore `powersave` first, then write the EPP preference selected by PPD for the profile and power state.
- No extra systemd service, D-Bus monitor, kernel boot parameter, or thermal configuration is added.

When the active `intel_pstate` performance algorithm owns EPP, the kernel may reject a non-performance EPP write with `EBUSY`. The patch avoids that conflict by ordering writes within PPD's existing profile activation flow.

## Key C source and upstream changes

The key implementation is available directly as [`src/ppd-driver-intel-pstate.c`](src/ppd-driver-intel-pstate.c). It is the complete modified Intel P-State driver source file based on PPD 0.30-2, with the upstream copyright and GPL-3 header retained. The complete change, including the related integration-test updates, is in [`intel-pstate-performance-governor.patch`](intel-pstate-performance-governor.patch). The `source/` directory also contains the Ubuntu 0.30-2 upstream tarball, Debian packaging tarball, and `.dsc` source-package descriptor used as the source base for the patched package.

This fix requires **modifying and rebuilding the upstream power-profiles-daemon source**. The patch changes governor/EPP application order in `src/ppd-driver-intel-pstate.c` and updates the assertions in `tests/integration_test.py` for performance and switching back. It is not a standalone script or sysfs setting. To unpack the source and apply the patch:

```sh
dpkg-source -x source/power-profiles-daemon_0.30-2.dsc
cd power-profiles-daemon-0.30
patch -p1 < ../intel-pstate-performance-governor.patch
dch --local xing1 "Use the intel_pstate performance governor for the performance profile"
dpkg-buildpackage -b -uc -us
```

The source descriptor (`.dsc`) carries an upstream signature. When extracting it with `dpkg-source`, confirm that the signature is verified by a trusted Debian/Ubuntu key configured on the machine. If the tool reports a missing key or an unverifiable signature, signature verification has not succeeded. `sha256sum -c SHA256SUMS` checks files against the repository manifest, but the manifest itself is unsigned and does not independently authenticate their origin. Then install the resulting `0.30-2xing1` package using the normal Debian packaging workflow and follow the verification steps above. These commands describe the source modification path; some build environments may need PPD build dependencies installed first.

## Build and verification record

- Source base: Ubuntu `power-profiles-daemon 0.30-2`.
- Debian package version: `0.30-2xing1`.
- Recorded local PPD test suite: **127 passed, 0 failed**.
- A live `balanced → performance → balanced` round trip was performed on the machine above. The performance profile had 16 `performance` governors. After returning to balanced on AC, all 16 governors were `powersave` and all 16 EPP values were `balance_performance`. No matching `busy`, `failed`, `error`, or `warning` entries were found in the checked PPD journal interval.
- No controlled benchmark was run, so no specific throughput, clock speed, or performance gain is claimed.

## Install

Download the patched package from the [GitHub Release `v0.30-2xing1`](https://github.com/xingwangzhe/ubuntu-intel-pstate-performance-governor/releases/tag/v0.30-2xing1). The release also includes the original Ubuntu `0.30-2` package for rollback. Installation replaces the system PPD package and changes system power policy. Review the package metadata and confirm its dependencies match your system first.

```sh
sha256sum -c SHA256SUMS
pkexec dpkg -i ./power-profiles-daemon_0.30-2xing1_amd64.deb
```

After installation, verify the active profile and every policy:

```sh
powerprofilesctl set performance
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c

powerprofilesctl set balanced
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c
for f in /sys/devices/system/cpu/cpufreq/policy*/energy_performance_preference; do cat "$f"; done | sort | uniq -c
journalctl -u power-profiles-daemon --since '10 minutes ago' --no-pager | grep -Ei 'busy|failed|error|warning'
```

On the verified AC setup, expect 16 `performance` governors in performance mode; in balanced mode, expect 16 `powersave` governors and 16 `balance_performance` EPP values. No output from the final command only means no matching log entries were found in that time interval.

## Rollback

```sh
pkexec dpkg -i ./power-profiles-daemon_0.30-2_amd64.deb
powerprofilesctl set balanced
```

## Limits and risk

The `performance` governor selects the driver's performance algorithm; it does not pin the CPU to its advertised maximum clock or override firmware power limits, thermal limits, or throttling. This package changes privileged system power policy and is not distributed through Ubuntu repositories. Review the patch and package metadata before use, and keep the original package or another recovery path available.

## Source and license

The PPD daemon and the C file here are licensed under GPL-3 as declared by upstream. The corresponding upstream source, Debian packaging, and local changes are provided together so the binary package can be reviewed and rebuilt. The patch also modifies `tests/integration_test.py`, which upstream `debian/copyright` identifies as GPL-2-or-later. Original copyright notices are retained; attribution to this project's local modifications does not claim ownership of upstream code or other contributors' work. License texts are in [`LICENSES/GPL-3.0.txt`](LICENSES/GPL-3.0.txt), [`LICENSES/GPL-2.0.txt`](LICENSES/GPL-2.0.txt), and [`LICENSES/GFDL-1.3.txt`](LICENSES/GFDL-1.3.txt). For file-by-file licensing and the applicable license versions, consult `debian/copyright` in the included source package.
