# Minecraft 冷备份系统部署、验收与交接手册

本手册适用于正式服“你好新蒸程”。所有命令默认在 Ubuntu 服务器以 root 执行。任何密码、Token、
授权码都不得写入交接记录或聊天。

## 一、固定环境

```text
Minecraft服务： hello-new-generation.service
服务目录：      /data/minecraft/hello-new-generation
运行用户：      minecraft:minecraft
用户HOME：      /data/minecraft
本地备份：      /data/minecraft/backups/hello-new-generation
RCON：          127.0.0.1:25575（不得公网映射）
```

## 二、源码清单

| 源文件 | 安装位置 | 用途 |
|---|---|---|
| `bin/minecraft-rcon.py` | `/usr/local/libexec/minecraft-backup/` | 本机 RCON 查询 |
| `bin/configure-rcon.py` | 同上 | 幂等配置 RCON |
| `bin/bdpan-listing.py` | 同上 | 解析并验证网盘 JSON 清单 |
| `bin/minecraft-backup.sh` | 同上 | 只归档已经停止的 Minecraft |
| `bin/minecraft-backup-upload.sh` | 同上 | 玩家检查、停服、恢复和上传编排 |
| `bin/minecraft-backup-recover.sh` | 同上 | systemd 异常收尾时恢复 Minecraft |
| `bin/minecraft-backup-now.sh` | `/usr/local/sbin/minecraft-backup-now` | 状态、日志、校验、云端查看、条件检查及安全启停入口 |
| `etc/minecraft-backup.conf` | `/etc/minecraft-backup.conf` | 非秘密配置 |
| `systemd/minecraft-backup-upload.service` | `/etc/systemd/system/` | 冷备份与上传服务 |
| `systemd/minecraft-backup-upload.timer` | `/etc/systemd/system/` | 上次检查结束 5 分钟后再检查 |

旧的 `minecraft-backup.service` 和 `minecraft-backup.timer` 属于已放弃的小时热备份方案，安装升级时会
被停用并删除。

## 三、为什么必须冷备份

正式服测试发现 `world/level.dat` 在 tar 读取期间变化。只读反编译进一步确认：Sable 的 paused 命令
只暂停物理；Civillis 使用独立异步 IO 队列，没有运行中 flush 管理命令。调用 Civil 的 close 会永久
关闭存储并最多等待队列 5 秒，不能作为热备份暂停接口。

因此不得添加 tar 忽略警告参数，不得排除 `level.dat` / `civil` 数据，不得恢复小时热备份 timer。

## 四、部署前只读检查

```bash
date --iso-8601=seconds
systemctl is-active hello-new-generation.service
systemctl cat hello-new-generation.service
systemctl show hello-new-generation.service \
  -p MainPID -p User -p Group -p WorkingDirectory \
  -p TimeoutStopUSec -p KillSignal -p SendSIGKILL --no-pager
df -hT /data
```

必须确认正式服由 systemd 管理、工作目录正确，并且已有正常停止流程或 Java 能响应 SIGTERM 关闭钩子。
如果 `systemctl stop` 只是立即 SIGKILL，不得部署自动冷备份。

再确认 RCON 与玩家：

```bash
python3 /usr/local/libexec/minecraft-backup/minecraft-rcon.py \
  --host 127.0.0.1 --port 25575 \
  --password-file /etc/minecraft-backup/rcon-password \
  --timeout 30 list
```

## 五、搬运和安装

Windows 仓库根目录：

```powershell
scp -P 56902 -r .\ops\backup root@wzk.rainplay.cn:/root/minecraft-backup-deploy
```

服务器：

```bash
cd /root/minecraft-backup-deploy
find . -maxdepth 3 -type f -printf '%P\n' | sort
bash -n install.sh bin/*.sh
python3 -m py_compile bin/*.py
python3 -m unittest discover -s tests -v
bash install.sh
```

不要使用 `--enable-timer`。普通安装只复制代码、迁移掉旧热备份单元并 daemon-reload，不重启 Minecraft。

## 六、安装后静态验收

```bash
systemctl is-active hello-new-generation.service
systemctl is-enabled minecraft-backup-upload.timer || true
systemctl is-active minecraft-backup-upload.timer || true
systemctl cat minecraft-backup-upload.service
systemctl cat minecraft-backup-upload.timer
test ! -e /etc/systemd/system/minecraft-backup.timer
test ! -e /etc/systemd/system/minecraft-backup.service
grep -E '^(MINECRAFT_SERVICE|MINECRAFT_USER|MINECRAFT_HOME|MINECRAFT_START_TIMEOUT_SECONDS|PLAYER_CHECK_MAX_GAP_SECONDS|UPLOAD_RETRY_DELAY_MINUTES|UPLOAD_PART_SIZE_MIB|REMOTE_QUERY_ATTEMPTS|REMOTE_QUERY_RETRY_SECONDS)=' \
  /etc/minecraft-backup.conf
```

预期 Minecraft 为 active；新 timer 为 disabled/inactive；旧热备份单元不存在；编排服务为
`User=root`，但脚本中的 tar 和 bdpan 路径通过 `runuser` 以 minecraft 执行。

现有旧热备份的 `.ready` 可以保留。新上传器只匹配 `_cold_*.ready`，不会上传旧热备份。如需明显标注：

```bash
cd /data/minecraft/backups/hello-new-generation
for marker in hello-new-generation_*.ready; do
  [[ -e "$marker" ]] || continue
  [[ "$marker" == *_cold_* ]] && continue
  mv -- "$marker" "${marker%.ready}.hot-unverified"
done
```

这只改变队列标记，不删除归档或 SHA-256。

## 七、首次冷备份测试前检查

首次测试会真实停止并启动 Minecraft。必须在批准的维护窗口、0 人在线时进行：

```bash
python3 /usr/local/libexec/minecraft-backup/minecraft-rcon.py \
  --host 127.0.0.1 --port 25575 \
  --password-file /etc/minecraft-backup/rcon-password \
  --timeout 30 list
systemctl is-active minecraft-backup-upload.service
systemctl is-active minecraft-backup-upload.timer
df -hT /data
```

两个备份单元必须 inactive，玩家必须为 0。归档阶段磁盘至少能容纳源目录加 5 GiB 保留空间；云端
上传还会临时生成一套等同归档大小的 256 MiB 分卷，成功后自动删除分卷。

清除旧测试计时：

```bash
rm -f \
  /data/minecraft/backups/hello-new-generation/.offline-since \
  /data/minecraft/backups/hello-new-generation/.offline-snapshot-for \
  /data/minecraft/backups/hello-new-generation/.cold-snapshot-for \
  /data/minecraft/backups/hello-new-generation/.last-player-check
```

## 八、完整 30 分钟验收

开始新计时并临时启动 timer：

```bash
minecraft-backup-now upload
systemctl start minecraft-backup-upload.timer
systemctl list-timers minecraft-backup-upload.timer --no-pager
```

必须看到 `starting the 30-minute quiet period`，且 `NEXT` 为约 5 分钟后的具体时间。使用第二个 SSH
窗口观察：

```bash
journalctl -fu minecraft-backup-upload.service
```

达到 30 分钟时预期顺序：

1. 最终确认仍为 0 人；
2. 日志显示停止正式服；
3. 服务变为 inactive、MainPID=0；
4. 产生 `_cold_` 归档并完成 tar、SHA-256 验证；
5. Minecraft 被启动；
6. 服务 active 且 RCON 可用；
7. 无人加入时把归档拆为 256 MiB 分卷并逐卷 bdpan 上传；
8. 最后上传 `.parts.sha256` 完成清单；
9. 日志出现 `Cloud multipart upload verified`，`.ready` 变为 `.uploaded`，本地临时分卷被删除。

另开窗口可观察正式服状态：

```bash
watch -n 2 'systemctl is-active hello-new-generation.service; systemctl show hello-new-generation.service -p MainPID --value'
```

不要手动 start/restart Minecraft，除非日志明确显示自动恢复失败。

## 九、失败恢复验收

任何失败后先执行：

```bash
systemctl stop minecraft-backup-upload.timer
systemctl is-active hello-new-generation.service
systemctl show minecraft-backup-upload.service \
  -p Result -p ExecMainStatus -p ActiveState -p SubState --no-pager
journalctl -u minecraft-backup-upload.service -n 300 --no-pager
```

如果 Minecraft 不是 active：

```bash
systemctl start hello-new-generation.service
systemctl is-active hello-new-generation.service
journalctl -u hello-new-generation.service -n 200 --no-pager
```

此时不得立即反复上传或启用 timer。保留日志、`.partial`、`.ready`、`.part-*` 和校验清单用于排查。
归档失败会删除 partial；上传失败会保留已生成分卷，默认退避 60 分钟，下一次只上传网盘缺失分卷，
不会再次关服生成冷备份。

分卷续传不得仅依据父目录单页列表。脚本会对每个分卷的完整远端路径做两次精确查询：名称与大小
一致才跳过；两次都明确返回“目录不存在”才上传；查询失败、异常 JSON 或大小冲突均停止并退避。
云端备份计数按 `--start/--limit 1000` 分页读取。

百度接口偶发 `context deadline exceeded` 或返回错误对象时，脚本不会把它解释为“不存在”。目录统计
和精确路径查询默认每隔 10 秒重试，最多 4 次；全部失败后才进入 60 分钟上传退避。

## 十、归档和网盘验收

```bash
cd /data/minecraft/backups/hello-new-generation
archive="$(find . -maxdepth 1 -type f \
  \( -name 'hello-new-generation_cold_*.tar.zst' -o -name 'hello-new-generation_cold_*.tar.gz' \) \
  -printf '%T@ %p\n' | sort -n | tail -n 1 | cut -d' ' -f2-)"
test -n "$archive"
sha256sum -c "${archive}.sha256"
tar -tf "$archive" >/dev/null
tar -tf "$archive" | grep -E '/(server.properties|world/level.dat)$'
test -f "${archive}.uploaded"
sudo -u minecraft -H /data/minecraft/.local/bin/bdpan \
  ls mc-backups/hello-new-generation
```

还需在百度网盘网页确认同名前缀的所有 `part-*`、归档 `.sha256` 与最后的 `.parts.sha256`，并确认
Minecraft 客户端可以重新进入、世界与 Sable/Civil 数据正常。

从云端下载到同一目录后，先重组再验归档：

```bash
sha256sum -c hello-new-generation_cold_*.tar.zst.parts.sha256
cat hello-new-generation_cold_*.tar.zst.part-* > restored.tar.zst
sed 's#  .*#  restored.tar.zst#' hello-new-generation_cold_*.tar.zst.sha256 |
  sha256sum -c -
tar -tf restored.tar.zst >/dev/null
```

## 十一、正式启用

只有首次完整冷备份、自动恢复和云端验证全部成功后：

```bash
cd /root/minecraft-backup-deploy
bash install.sh --enable-timer
systemctl is-enabled minecraft-backup-upload.timer
systemctl list-timers minecraft-backup-upload.timer --no-pager
```

不再存在小时本地 timer。每轮玩家从“有人”变为连续无人 30 分钟时只生成一份冷备份。

## 十二、日常维护、停用与回滚

常用只读维护入口：

```bash
minecraft-backup-now status
minecraft-backup-now logs 200
minecraft-backup-now verify-local
minecraft-backup-now cloud
```

`verify-local` 会完整读取本地冷备份并核对 SHA-256，不会停服；`cloud` 只读访问
`/apps/bdpan/mc-backups/hello-new-generation`。日常至少检查 Minecraft 为 active、timer 为
enabled/active、没有长期滞留的 `.partial`，并确认最新成功日志含 `Cloud multipart upload verified`。

### 已验证的停服替换式本地回档

`2026-07-17` 已在正式服用
`hello-new-generation_cold_2026-07-17_01-26-36_+0800.tar.zst` 完成一次回档。所有玩家下线后，正式运行
存档被目标备份完整替换。为使操作可撤回，归档没有直接解压进运行目录，而是先解压、检查和配置暂存
副本，停服后才通过目录改名切换；切换前的正式目录另存为
`/data/minecraft/hello-new-generation.before-rollback-20260717-114542`。

重复执行时先建立本次变量，归档名必须由操作员明确填写，不得用“最新文件”模糊选择：

```bash
archive=/data/minecraft/backups/hello-new-generation/hello-new-generation_cold_YYYY-MM-DD_HH-MM-SS_+0800.tar.zst
service=hello-new-generation.service
live=/data/minecraft/hello-new-generation
staging=/data/minecraft/hello-new-generation.restore-staging
safety="${live}.before-rollback-$(date '+%Y%m%d-%H%M%S')"
```

先暂停未来调度，并在 Minecraft 仍运行时完成只读校验与独立解压：

```bash
minecraft-backup-now pause
archive_dir="$(dirname "$archive")"
archive_name="$(basename "$archive")"
(cd "$archive_dir" && sha256sum -c "${archive_name}.sha256")
tar -tf "$archive" >/dev/null
tar -tf "$archive" | grep -E '^hello-new-generation/(server.properties|world/level.dat)$'

test ! -e "$staging"
install -d -o minecraft -g minecraft -m 0750 "$staging"
runuser -u minecraft -- tar --zstd -xf "$archive" \
  -C "$staging" --strip-components=1
test -s "$staging/world/level.dat"
```

归档中的 `server.properties` 可能含旧 RCON 密码。切换前必须使用现有受保护密码文件更新暂存副本，
不得把密码输出到终端或文档：

```bash
python3 /usr/local/libexec/minecraft-backup/configure-rcon.py \
  "$staging/server.properties" \
  /etc/minecraft-backup/rcon-password
chown minecraft:minecraft "$staging/server.properties"
chmod 0640 "$staging/server.properties"
```

让所有玩家下线并通过 RCON 再次确认人数为 0 后，才进入停服切换窗口：

```bash
python3 /usr/local/libexec/minecraft-backup/minecraft-rcon.py \
  --host 127.0.0.1 \
  --port 25575 \
  --password-file /etc/minecraft-backup/rcon-password \
  --timeout 30 \
  list

systemctl stop "$service"
test "$(systemctl show "$service" -p MainPID --value)" = 0
test ! -e "$safety"
mv "$live" "$safety"
mv "$staging" "$live"
systemctl start "$service"

restore_ready=0
for _ in $(seq 1 60); do
  if systemctl is-active --quiet "$service" && \
     python3 /usr/local/libexec/minecraft-backup/minecraft-rcon.py \
       --host 127.0.0.1 \
       --port 25575 \
       --password-file /etc/minecraft-backup/rcon-password \
       --timeout 10 \
       list >/dev/null 2>&1; then
    restore_ready=1
    break
  fi
  sleep 5
done
test "$restore_ready" = 1
```

启动后必须确认服务 active、RCON 可用，再由玩家验证位置、背包、末影箱、世界、Sable 子世界、Civil
数据、Create 机器、任务和权限。若最后的启动检查或客户端验收失败，保持 timer 停用并执行：

```bash
systemctl stop "$service"
test "$(systemctl show "$service" -p MainPID --value)" = 0
failed="${live}.failed-after-rollback-$(date '+%Y%m%d-%H%M%S')"
test -e "$safety"
if test -e "$live"; then
  test ! -e "$failed"
  mv "$live" "$failed"
fi
mv "$safety" "$live"
systemctl start "$service"
```

随后重新验证服务与 RCON。不要覆盖、合并或删除任一目录；失败目录也要保留到原因查明。

本次生产演练中服务成功出现 `Done`，RCON 恢复，玩家登录和实际游玩无异常。验收后执行
`minecraft-backup-now resume`，当时状态显示 timer 为 enabled/active、Minecraft 为 active、2 名玩家
在线、`Offline since: none`，证明观察期从零开始且没有在玩家在线时停服。

本次只验证了本地完整归档的停服替换式回档。仅依赖百度网盘的下载、分卷校验、重组与回档仍是未验收边界。
回档前安全目录至少保留到产生并验证一份回档后的新冷备份；清理属于独立维护操作，不得顺手执行。

### bdpan 登录失效

必须用 `minecraft` 用户和 `/data/minecraft` HOME；用 root 执行会误用 `/root/.config/bdpan`。先暂停
自动化，再获取授权链接：

```bash
minecraft-backup-now pause
runuser -u minecraft -- env \
  HOME=/data/minecraft \
  PATH=/data/minecraft/.local/bin:/usr/bin:/bin \
  /data/minecraft/.local/bin/bdpan \
  login --get-auth-url --accept-disclaimer
```

在本人浏览器完成授权。不得把授权码、Token 或配置文件发到聊天。用 stdin 安全提交授权码：

```bash
read -rsp 'bdpan授权码: ' BDPAN_CODE; echo
printf '%s\n' "$BDPAN_CODE" | runuser -u minecraft -- env \
  HOME=/data/minecraft \
  PATH=/data/minecraft/.local/bin:/usr/bin:/bin \
  /data/minecraft/.local/bin/bdpan \
  login --set-code-stdin --accept-disclaimer
unset BDPAN_CODE
runuser -u minecraft -- env HOME=/data/minecraft \
  /data/minecraft/.local/bin/bdpan whoami
minecraft-backup-now resume
```

Token 配置只允许保存在 `/data/minecraft/.config/bdpan/config.json`。

暂停未来调度但不删除备份、不打断当前任务：

```bash
minecraft-backup-now pause
```

维护完成后安全恢复；该命令会做 RCON/bdpan 前置检查，并从零重新计算 30 分钟：

```bash
minecraft-backup-now resume
```

不要把 `systemctl stop minecraft-backup-upload.service` 当作普通暂停命令。如果确有紧急情况，先用
`minecraft-backup-now status` 和日志确认任务阶段；强行停止会触发退出陷阱与 `ExecStopPost`，操作后
必须亲自确认 `hello-new-generation.service` 为 active。

RCON 本身无需因停用备份而撤回。如果确需撤回，必须另行安排维护窗口并按已保存的
`server.properties.before-backup-*` 谨慎恢复，不要整体覆盖后来产生的其他配置修改。

## 十三、交接完成清单

带 `[x]` 的项目已有本次生产证据；未勾选项必须继续保留为已知待验收边界，不得口头宣称完成。

- [x] 正式服务的停止方式确认是正常关闭而非立即 SIGKILL；
- [ ] RCON 只监听/使用本机，公网未映射 25575；
- [x] 旧小时热备份 service/timer 已移除；
- [x] 旧热备份没有被上传为冷备份；
- [x] 连续离线计时与 10 分钟检查中断重置已验证；
- [x] 0 人时自动停止 Minecraft 已验证；
- [x] 冷备份归档、SHA-256 和关键文件已验证；
- [ ] 压缩成功和失败路径都能恢复 Minecraft；
- [x] Minecraft 重启后 RCON 和客户端登录正常；
- [x] 所有玩家下线后，本地冷归档已完成一次停服替换式生产回档并由玩家上线验证；
- [ ] 仅依赖云端分卷的下载、重组与完整回档；
- [x] bdpan 网页与命令行均看到全部分卷、归档 SHA-256 和最后写入的分卷清单；
- [ ] 本地上限 3、云端硬上限 100；
- [x] 唯一 timer 已启用并记录下一次检查时间；
- [x] 负责人知道如何停用 timer、人工恢复 Minecraft 并从零恢复观察期。
