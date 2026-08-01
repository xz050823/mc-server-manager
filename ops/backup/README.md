# Minecraft 连续离线冷备份与百度网盘上传

这套部署包用于正式服“你好新蒸程”：Ubuntu 24.04、NeoForge 1.21.1、服务目录
`/data/minecraft/hello-new-generation`、systemd 服务 `hello-new-generation.service`。不增加
Minecraft 模组。

完整部署、首次验收、故障恢复和交接步骤见 [HANDOFF.md](HANDOFF.md)。

## 当前状态（2026-07-17）

正式服已验证 RCON、bdpan 登录和一份 3 GB 热备份，但热备份测试发现 Sable 与 Civillis 会在
`save-off` / `save-all flush` 后继续或异步写入世界目录。两次冲突都被 tar 严格检测并安全中止，
没有上传可疑归档，Minecraft 自动保存和服务均已恢复。

因此已经放弃小时热备份。首次冷备份在 `2026-07-13 06:17 UTC` 成功：停服归档 47 秒，Minecraft
随后自动启动并恢复 RCON。直接上传整个 3 GB 文件失败后改为 256 MiB 分卷；续传期间经历 HTTP 504、
上传失败和远端查询超时，脚本均保留本地数据、跳过已验证分卷且没有重复停服。

`2026-07-13 10:06 UTC` 首份冷备份完成全部 12 个分卷、归档 SHA-256 与最终 `.parts.sha256` 上传，
日志为 `Cloud multipart upload verified (1/100)`；本地 `.ready` 已转为 `.uploaded`，临时分卷已清理，
原归档 SHA-256 复核通过。现有旧热备份文件不带 `_cold_`，上传器不会选择它。

正式 timer 已于 `2026-07-13 10:11 UTC` 启用；第一次自动检查在 `10:16:50 UTC` 正常触发，确认
0 人在线后只开始新的 30 分钟安静期，Minecraft 保持 active，下一次 5 分钟检查已排程。

`2026-07-17` 已用本地冷备份
`hello-new-generation_cold_2026-07-17_01-26-36_+0800.tar.zst` 完成一次停服替换式生产回档：所有玩家
下线后，正式运行存档由目标备份完整替换；为保留撤回能力，归档先完成 SHA-256 与 tar 结构校验并
解压到独立暂存目录，停服后把回档前正式目录改名保留为
`/data/minecraft/hello-new-generation.before-rollback-20260717-114542`，最后才切换恢复目录。服务启动、
RCON、玩家上线连接和实际游玩均正常。验收期间 timer 保持停用，确认无误后通过
`minecraft-backup-now resume` 从零重新计算 30 分钟观察期；恢复时有 2 名玩家在线，因此没有误触发停服。

## 最终策略

- 每 5 分钟通过本机 RCON 查询在线人数；
- 第一次发现 0 人只开始计时；
- 每次检查发现玩家在线，立即清空离线计时；
- 检查中断超过 10 分钟，也清空旧计时，避免把不可观察的时间算作连续离线；
- 连续 30 分钟检查均无人在线后，再做一次最终 RCON 人数确认；
- 正常停止 `hello-new-generation.service`，等待 Java 和模组关闭保存流程完成；
- 在 Minecraft 完全停止、`MainPID=0` 后制作一份完整冷备份；
- 启动 Minecraft，等待服务变为 active 且 RCON 恢复；
- 再次确认没有玩家后，把冷备份拆为 256 MiB 分卷，逐卷校验并上传百度网盘；
- 每轮连续离线只制作一次冷备份；本地最多保留 3 份，云端硬上限 100 份。

玩家可能在最终人数检查和端口关闭之间尝试登录，这是无法完全消除的短暂竞态。正常情况下其客户端
只会看到服务器暂时不可用，约 2–3 分钟后可重新连接。脚本不会在检测到已有在线玩家时主动停服。

## 为什么不再热备份

只读调查确认：

- Sable 1.2.2 大约每五分钟参与一次子世界保存；
- `/sable paused` 只调用 `SubLevelPhysicsSystem.setPaused()`，仅暂停物理，不暂停存储；
- Civillis 2.0.0 使用独立的 `ColdIOQueue`、异步 SaveRequest 和 NBT 存储；
- Civil 管理命令只有 `rebuild` 与 `ring`，没有 save、flush 或 pause；
- `NbtStorage.close()` 会关闭存储并最多等待队列 5 秒，只适用于正常关闭流程，不能在运行中调用后恢复。

因此忽略 `tar: file changed as we read it`、排除 `level.dat` 或不断增加等待时间都不安全。冷备份让
Minecraft 和模组按正常关闭流程结束写入，再读取静止的目录。

## 安全设计

- 冷备份编排服务以 root 运行，只有它负责停启 systemd 服务；
- tar 和 bdpan 均通过 `runuser` 降权为 `minecraft` 用户执行；
- 停服前写入 `/run/minecraft-backup/minecraft-stopped-by-backup`；
- 正常路径、错误退出和 systemd `ExecStopPost` 都会根据该标记尝试恢复 Minecraft；
- Minecraft 未完全停止时，归档脚本拒绝运行；
- Minecraft 未恢复 active 且 RCON 未恢复时，不开始云端上传；
- 上传期间每分钟观察一次玩家，若发现玩家，本轮上传完成后重置下一轮离线计时；
- 上传期间每分钟刷新检查连续性，timer 从任务结束后再等待 5 分钟，不会在长任务结束时立即补跑；
- 归档先写 `.partial`，完整读取验证后才改为正式文件并生成 `.sha256`、`.ready`；
- 只有 `${BACKUP_PREFIX}_cold_*.ready` 会进入上传队列；旧热备份不会误上传；
- 已有 `.ready` 时绝不再次关服；上传失败默认退避 60 分钟，只续传网盘缺失的 256 MiB 分卷；
- 每个分卷上传前对完整远端路径查询两次；只有明确返回“不存在”才上传，异常或大小冲突一律停止；
- 远端目录或精确文件查询遇到超时、错误对象等不安全结果时，默认每隔 10 秒重试，最多 4 次；重试期间绝不把查询失败当作文件缺失；
- 云端上限统计按 1000 项分页遍历，不依赖单页父目录结果；
- 云端最后上传 `.parts.sha256`，它是该备份所有分卷均已验证的完成标记；
- 本地备份、停服编排和上传共用一把 `flock`，禁止并发执行。

## 文件位置

```text
服务器目录：/data/minecraft/hello-new-generation
本地备份：  /data/minecraft/backups/hello-new-generation
配置：      /etc/minecraft-backup.conf
RCON密码：  /etc/minecraft-backup/rcon-password
远端目录：  /apps/bdpan/mc-backups/hello-new-generation/
网页显示：  我的应用数据/bdpan/mc-backups/hello-new-generation/
```

冷备份文件示例：

```text
hello-new-generation_cold_2026-07-13_12-34-56_+0800.tar.zst
hello-new-generation_cold_2026-07-13_12-34-56_+0800.tar.zst.sha256
hello-new-generation_cold_2026-07-13_12-34-56_+0800.tar.zst.uploaded
```

云端保存 `...tar.zst.part-000`、`part-001` 等分卷、完整归档的 `.sha256`，以及最后写入的
`.parts.sha256` 完成清单；不会再直接上传 3 GB 单文件。

## 从 Windows 上传源码

在仓库根目录执行：

```powershell
scp -P 56902 -r .\ops\backup root@wzk.rainplay.cn:/root/minecraft-backup-deploy
```

然后在服务器中安装，但首次不要启用 timer：

```bash
cd /root/minecraft-backup-deploy
bash -n install.sh bin/*.sh
python3 -m py_compile bin/*.py
bash install.sh
```

安装器会停用并删除旧的 `minecraft-backup.timer` 与 `minecraft-backup.service`，只安装冷备份上传服务和
一个 5 分钟检查 timer。它不会重启 Minecraft，也不会自动启用 timer。

## 管理员命令

```bash
minecraft-backup-now status
minecraft-backup-now logs 200
minecraft-backup-now verify-local
minecraft-backup-now cloud
minecraft-backup-now upload
minecraft-backup-now pause
minecraft-backup-now resume
```

- `status`：只读显示 Minecraft、玩家、timer、最近结果、离线/重试状态、本地标记与磁盘空间；
- `logs [行数]`：只读显示最近日志，默认 200 行，最多 5000 行；
- `verify-local`：重新计算所有本地冷备份 SHA-256。不会停服，但会产生数十秒磁盘读取；
- `cloud`：以 `minecraft` 用户只读列出百度网盘备份目录；
- `upload` / `check`：执行一次正常条件检查，不绕过在线人数或连续离线 30 分钟。若条件已经满足且
  没有 `.ready`，会真实停服冷备份；若已有 `.ready`，只处理上传；
- `pause`：停用未来自动检查，但不强行中止正在进行的冷备份或上传，以便恢复路径正常收尾；
- `resume`：检查 Minecraft、RCON 和 bdpan 登录，清除旧观察计时并启用 timer，重新计满 30 分钟；
- 旧的 `local` 热备份命令已删除。

## 关键配置

```text
MAX_LOCAL_BACKUPS=3
MIN_FREE_GIB=5
OFFLINE_DELAY_MINUTES=30
PLAYER_CHECK_MAX_GAP_SECONDS=600
UPLOAD_RETRY_DELAY_MINUTES=60
UPLOAD_PART_SIZE_MIB=256
REMOTE_QUERY_ATTEMPTS=4
REMOTE_QUERY_RETRY_SECONDS=10
MINECRAFT_START_TIMEOUT_SECONDS=300
REMOTE_MAX_BACKUPS=100
```

## 日常检查

```bash
minecraft-backup-now status
minecraft-backup-now logs 200
```

建议每周执行一次 `verify-local`，并用 `cloud` 或百度网盘网页确认最新备份存在。看到 `.ready` 表示
等待上传或续传；`.uploaded` 表示云端完成清单已经验证；`.partial` 表示归档中断，应立即查看日志。

维护暂停和恢复使用：

```bash
minecraft-backup-now pause
# 完成维护后
minecraft-backup-now resume
```

`pause` 不会杀掉正在运行的任务。只有明确确认自动恢复失败、且理解当前所处阶段时，才直接操作
`minecraft-backup-upload.service`；之后必须检查 `hello-new-generation.service` 为 active。

## bdpan 账号维护

bdpan 必须始终以 `minecraft` 用户、`HOME=/data/minecraft` 运行；不要用 root 登录，否则 Token 会写到
错误的 `/root/.config/bdpan`。先检查：

```bash
runuser -u minecraft -- env \
  HOME=/data/minecraft \
  PATH=/data/minecraft/.local/bin:/usr/bin:/bin \
  /data/minecraft/.local/bin/bdpan whoami
```

若登录失效，先执行 `minecraft-backup-now pause`，再运行：

```bash
runuser -u minecraft -- env \
  HOME=/data/minecraft \
  PATH=/data/minecraft/.local/bin:/usr/bin:/bin \
  /data/minecraft/.local/bin/bdpan \
  login --get-auth-url --accept-disclaimer
```

在自己的浏览器完成授权，授权码不要发到聊天或写入文件。回到 SSH 后安全输入：

```bash
read -rsp 'bdpan授权码: ' BDPAN_CODE; echo
printf '%s\n' "$BDPAN_CODE" | runuser -u minecraft -- env \
  HOME=/data/minecraft \
  PATH=/data/minecraft/.local/bin:/usr/bin:/bin \
  /data/minecraft/.local/bin/bdpan \
  login --set-code-stdin --accept-disclaimer
unset BDPAN_CODE
```

重新执行 `whoami`，成功后用 `minecraft-backup-now resume` 恢复自动化。Token 位于
`/data/minecraft/.config/bdpan/config.json`，不得复制到仓库、工单或聊天。

成功日志必须依次包含：

```text
The quiet period is complete; stopping ... for a consistent cold backup
Cold backup completed and queued for upload
... is active and RCON is ready
Cloud multipart upload verified
```

## 云端 100 份上限

官方 bdpan 没有删除命令。云端已有 100 个完成清单时，脚本保留本地 `.ready` 并暂停新增上传。
管理员在百度网盘网页或客户端删除某一旧备份的全部 `part-*`、`.sha256` 与 `.parts.sha256` 后，
后续检查会恢复上传。

## 恢复边界

恢复备份必须另行安排停服维护窗口。本地冷备份的停服替换式回档已经在生产环境验证：先暂停 timer，
校验 SHA-256 和 tar 结构，解压到独立暂存目录并同步当前 RCON 配置；确认所有玩家下线且 RCON 为 0 人
后停止服务，把回档前正式目录改名保留，再通过同一文件系统内的目录改名让目标备份成为新的正式
存档。启动失败时应立即把失败目录移开并把原目录换回。服务、RCON 和
客户端验证完成前不得恢复 timer，也不得删除旧目录。详细步骤与成功证据见 [HANDOFF.md](HANDOFF.md)。

若从云端恢复，仍需先按 `.parts.sha256` 验证全部分卷，再按文件名顺序
`cat ...part-* > ...tar.zst`，最后使用归档自身的 `.sha256` 做第二次校验。云端下载、分卷重组并切换
正式目录的端到端演练尚未执行，不能用本次本地回档替代该项结论。

## 尚需长期验证的边界

首次生产冷备份、恢复启动、本地归档停服替换式回档、分卷断点续传和自动 timer 已通过；以下边界已有
保护逻辑和文档，但尚未用生产数据主动触发：本地生成第 4 份时的“最多 3 份”清理、云端达到第 100
份时的硬停止、仅依赖云端分卷的完整灾难恢复，以及 bdpan Token 长期自动刷新。不要为了测试上限而
批量制造备份或删除正式存档。
