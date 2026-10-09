# 脱敏构建记录

这是对本地 PPD 构建信息的有限脱敏摘要，不是聊天记录，也不是完整构建日志。

- 源码基线：Ubuntu Resolute 的 `power-profiles-daemon 0.30-2` 源码包。
- 本地构建版本：`0.30-2xing1`，架构 `amd64`。
- 源码改动见仓库根目录的 `intel-pstate-performance-governor.patch`；修改后的 Intel P-State 驱动见 `src/ppd-driver-intel-pstate.c`。
- 构建元数据将环境标记为受本地安装的配置文件、库和程序影响（`usr-local-has-configs`、`usr-local-has-libraries`、`usr-local-has-programs`）。因此，本记录不声称该构建来自干净隔离或完全可复现的环境。
- 测试套件结果及实机模式往返验证已记入项目 README，并注明适用机器与验证范围。

原始 `.buildinfo`、`.changes`、Debian changelog 新增记录、解出的临时源码树、生成的二进制文件和本地 sysfs 切换辅助脚本均不纳入提交。原始元数据含个人联系信息或较详细的主机/构建环境信息；生成物和辅助脚本也不是审阅或重建源码改动所必需的。检查过的文件中没有发现聊天记录。
