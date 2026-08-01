# 长期 Minecraft 云服务器日志

## 2026-07-03

### 连接信息

- 用途：长期使用的新高性能云服务器
- SSH 命令：`ssh -p 56902 root@wzk.rainplay.cn`
- 主机：`wzk.rainplay.cn`
- SSH 端口：`56902`
- 用户名：`root`
- 密码：仅保存在本地 `.local/servers.yaml`，不进入 Git 仓库。

### 当前任务

- 先连接服务器。
- 查看基础环境。
- 暂不部署整合包，等待后续配置任务。

### SSH 连接检查

- DNS 解析：
  - `wzk.rainplay.cn -> 122.228.216.41`
- TCP 连通性：
  - `56902` 端口可连接
- SSH 主机指纹：
  - `ssh-ed25519 255 SHA256:Cnv7QmX5pg/NMTsLyClUTvPOaUAj23EX3wUtcHMNJH0`
- 已成功用 `root` 登录。

### 基础环境盘点

- 主机名：`RainYun-k99s4z3N`
- 系统：`Ubuntu 24.04.1 LTS`
- 内核：`Linux 6.8.0-49-generic`
- 架构：`x86_64`
- 运行时间：检查时约 `11 min`
- CPU：
  - 4 核
  - `AMD Ryzen 9 9950X 16-Core Processor`
  - KVM 虚拟化
- 内存：
  - 总内存约 `15 GiB`
  - 检查时可用约 `15 GiB`
  - Swap：`1.4 GiB`，未使用
- 磁盘：
  - 系统盘：`/dev/sda2`，ext4，30G，挂载到 `/`
  - 数据盘：`/dev/sdb1`，ext4，50G，UUID `d1edfdf2-2156-4230-8dae-3ef79bc85558`
  - `/etc/fstab` 中写有 `/dev/sdb1 /data ext4 defaults 0 2`
  - 但检查时 `df -hT` 未显示 `/data` 独立挂载，说明数据盘当前可能没有实际挂载
- Java：
  - 当前未安装，`java: command not found`
- 常用工具：
  - 已有：`apt`、`curl`、`wget`、`screen`、`tmux`
  - 未检测到：`java`、`zip`、`unzip`
- 运行状态：
  - 没有 Java/Minecraft 相关进程
  - 没有 Minecraft 相关 systemd 服务在运行
  - 当前监听端口主要是系统 DNS 和 SSH
- 防火墙：
  - `ufw` 状态：inactive
- 软件源：
  - Ubuntu 源使用中科大镜像：`http://mirrors.ustc.edu.cn/ubuntu/`
- 时区：
  - 当前为 `Etc/UTC`

### 初步判断

- 这是一台基本全新的 Ubuntu 服务器，适合从零搭建长期 Minecraft 整合包服。
- 性能明显高于上一台测试服：4 核、约 15 GiB 内存、50G 数据盘。
- 后续正式部署前建议先处理：
  - 挂载 `/dev/sdb1` 到 `/data`
  - 安装 Java 21 或按整合包版本安装对应 Java
  - 安装 `zip`、`unzip` 等打包工具
  - 创建专用 `minecraft` 用户
  - 使用 systemd 管理服务端

## 2026-07-03 你好新蒸程 1.5.9 服务端部署

### 本地服务端包

- 源文件：`整合包/你好新蒸程/服务端/你好新蒸程1.5.9.-Server .zip`
- 本地硬链接暂存：`staging/hello-new-generation-1.5.9-server.zip`
- 文件大小：`1044005501` bytes，约 `996M`
- SHA256：`514a8ec7f6aa566fd1328a1151b1786063e55a553dbd918bc9fd82ed239ec9bb`
- 压缩包条目数：`2538`
- 解压后约：`1.27G`
- mod jar 数：约 `318`；部署后启用 `223`，disabled `2`
- 服务端包自带：`run.sh`、`run.bat`、`user_jvm_args.txt`、`server.properties`、`world/`、`Oracle-jdk-21/`

### 服主必看内容摘要

- 物理化必须报备，航空学载具累计多会吃内存和 CPU。
- 物理载具容易消失时可以拉高服务器模拟距离，但前提是内存足够。
- 安装了 C2ME，成员跑图会占用全核资源；加载快，但容易卡顿。
- 直接在服务器创建存档没有作者建议的好看地形，建议从客户端按教程创建并导入。

### 远程环境准备

- 已挂载数据盘：`/dev/sdb1 -> /data`
- `/data` 容量：约 `49G`
- 已安装：
  - `openjdk-21-jre-headless`
  - `unzip`
  - `zip`
  - `curl`
  - `screen`
  - `tmux`
- Java 版本：`OpenJDK 21.0.11`
- 已创建运行用户：`minecraft`
- 已创建目录：
  - `/data/minecraft/hello-new-generation`
  - `/data/minecraft/uploads`
  - `/data/minecraft/backups`

### 上传与校验

- 上传方式：本地临时 HTTP + SSH 反向隧道，远程 curl 拉取。
- 远程压缩包：`/data/minecraft/uploads/hello-new-generation-1.5.9-server.zip`
- 远程 SHA256 校验通过：`514a8ec7f6aa566fd1328a1151b1786063e55a553dbd918bc9fd82ed239ec9bb`

### 部署目录

- 解压根目录自动识别为：压缩包内的双层 `你好新蒸程1.5.9-Server/你好新蒸程1.5.9-Server`
- 正式部署目录：`/data/minecraft/hello-new-generation`
- 部署目录大小：约 `1.3G`
- 已接受 EULA：`eula=true`
- 启动脚本：`/data/minecraft/hello-new-generation/start.sh`
- 原包启动脚本：`/data/minecraft/hello-new-generation/run.sh`
- JVM 参数沿用原包：
  - `-Xmx12G -Xms8G`
  - `ZGC`
  - `UseLargePages`
- 服务端关键配置沿用原包：
  - `server-port=25565`
  - `online-mode=true`
  - `difficulty=hard`
  - `enable-command-block=true`
  - `max-players=20`
  - `view-distance=10`
  - `simulation-distance=8`
  - `motd=HelloNewGeneration1.5.2 Minecraft Server`

### systemd 服务

- 服务名：`hello-new-generation.service`
- 服务文件：`/etc/systemd/system/hello-new-generation.service`
- 已启用开机自启：`systemctl enable hello-new-generation.service`
- 运行用户：`minecraft`
- 工作目录：`/data/minecraft/hello-new-generation`
- 启动命令：`/data/minecraft/hello-new-generation/start.sh`

### 服务端验证

- 验证方式：只做服务端日志、systemd 状态、端口监听检查；未尝试客户端连接。
- 启动结果：成功
- 日志出现：`Done (7.067s)! For help, type "help"`
- 45 秒后复查：仍为 `active (running)`
- 进程：`java @user_jvm_args.txt ... @libraries/net/neoforged/neoforge/21.1.233/unix_args.txt nogui`
- 内部监听端口：
  - TCP `*:25565`
  - UDP `*:25565`
- 资源占用：
  - systemd 显示内存约 `9.9G`
  - `/data` 使用约 `2.4G`，可用约 `45G`
- 日志中存在大量 DisplayDelight/菜品 registry warning，以及部分 namespace warning；服务端仍成功启动。

### 后续客户端验证

- 按用户要求，未进行客户端连接验证。
- 用户可在雨云控制台确认外部端口映射到服务器内部 `25565` 后，自行用客户端连接测试。

### 2026-07-03 OP 权限配置

- 玩家：`ye_fan_233`
- UUID：`07713d91-9a24-4266-a021-1644777bcc9d`
- 操作：短暂停止 `hello-new-generation.service`，备份并写入 `ops.json`，然后重启服务。
- OP 等级：`4`
- `bypassesPlayerLimit`：`false`
- 备份目录：`/data/minecraft/backups/op-ye_fan_233-20260702-180712`
- 写入后的 `ops.json`：

```json
[
  {
    "uuid": "07713d91-9a24-4266-a021-1644777bcc9d",
    "name": "ye_fan_233",
    "level": 4,
    "bypassesPlayerLimit": false
  }
]
```

- 验证结果：
  - `hello-new-generation.service` 为 `active`
  - TCP `*:25565` 正在监听
  - UDP `*:25565` 正在监听

### 2026-07-03 客户端导出包无法进服分析

- 用户现象：原客户端可正常加入并游玩；PCL2 导出后的客户端可启动、可单机游玩，但加入服务器到“加入世界中”阶段时显示大量 `ResourceKey[minecraft:item / hello_new_generation_core:*]`。
- 本次操作范围：只分析本地服务端压缩包和截图信息；未停止服务端，未修改远程服务端，未执行任何会干扰服务器运行的操作。
- 关键发现：服务端包内存在 `mods/hello_new_generation_core-0.0.15.jar`，截图中的命名空间 `hello_new_generation_core` 来自这个自定义核心模组。
- 关键文件：服务端包根目录存在 `hello_new_generation_core/ship_blueprint.txt`。
- `ship_blueprint.txt` 第一行包含截图里出现的 `aa6`、`c8`、`e13`、`b10`、`ss1` 等 ID，说明这些物品注册项来自该蓝图表。
- 字节码检查结论：核心模组会读取当前游戏目录下的 `hello_new_generation_core//ship_blueprint.txt`；如果该文件不存在，会创建空文件。文件为空或缺失时，客户端注册出的 `hello_new_generation_core:*` 物品会少于服务端，加入服务器时触发 NeoForge 注册表不一致。
- 初步判断：PCL2 整合包导出过程很可能没有包含非标准根目录 `hello_new_generation_core/`，或者导出后该文件位置不在游戏工作目录根部，导致导出客户端注册表与服务端不一致。
- 建议修复：在导出的客户端压缩包里补上 `hello_new_generation_core/ship_blueprint.txt`，位置应与 `mods/`、`config/`、`kubejs/` 同级；同时确认 `mods/hello_new_generation_core-0.0.15.jar` 存在且版本一致。

### 2026-07-03 导出客户端路径对比确认

- 正常客户端：`E:\Game\Minecraft\整合包\versions\你好，新蒸程V1.5.9`
- 异常导出客户端：`E:\Game\Minecraft\整合包导出测试temp\你好，新蒸程V1.5.9\.minecraft\versions\你好，新蒸程V1.5.9`
- 本次操作范围：只读取本地客户端目录和日志；未连接、未停止、未修改远程服务端。
- 两边核心模组一致：
  - 文件：`mods\hello_new_generation_core-0.0.15.jar`
  - 大小：`98365` 字节
  - SHA256：`28E416A12300B5685F7A6AC4E7FEB90BF0F7D21EDAA130FA662A17B591EEF012`
- 差异文件：`hello_new_generation_core\ship_blueprint.txt`
  - 正常客户端大小：`1717` 字节
  - 正常客户端 SHA256：`2A242542901766FE7CC343E16D42D89F36A0013432B0253AE99A6D91A8DD667C`
  - 异常导出客户端大小：`0` 字节
  - 异常导出客户端 SHA256：`E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855`（空文件）
- 异常客户端 `latest.log` 明确记录：`Client disconnected with reason: 服务器发送含有未知键的注册表：ResourceKey[minecraft:item / hello_new_generation_core:...]`。
- 结论：PCL2 导出包中的 `hello_new_generation_core/ship_blueprint.txt` 为空，导致客户端没有注册服务端拥有的蓝图物品；连接到“加入世界中”时 NeoForge 注册表同步失败。
- 修复方向：把正常客户端的 `hello_new_generation_core\ship_blueprint.txt` 复制到导出客户端相同位置，重新启动导出客户端后再进服测试。

### 2026-07-07 身份验证服务器宕机提示诊断

- 用户现象：多个客户端同时无法进入，卡在“通信加密中”，随后提示身份验证服务器处于宕机状态。
- 本次操作范围：只读检查远程服务端状态、日志、DNS 和出站连通性；未修改配置，未重启服务端。
- 服务端状态：`hello-new-generation.service` 为 `active`。
- 认证相关配置：
  - `online-mode=true`
  - `enforce-secure-profile=true`
- 日志表现：多个玩家在登录阶段被踢出，日志反复出现：
  - `Authentication servers are down. Please try again later, sorry!`
  - `Couldn't verify username because servers are unavailable`
- 网络检查：
  - 到公网 IP `1.1.1.1`、`8.8.8.8` ping 可达，说明基础出站网络存在。
  - 服务器系统 DNS 通过 `systemd-resolved` 使用 `223.5.5.5`。
  - `sessionserver.mojang.com`、`api.minecraftservices.com`、`authserver.mojang.com` 使用系统解析均超时。
  - 通过域名访问 Mojang/Microsoft 认证接口时也卡在解析阶段。
- 判断：服务端运行正常，问题集中在云服务器出站 DNS/认证接口解析链路；多个客户端同时失败符合服务端无法访问 Mojang/Microsoft 会话服务器的现象。
- 可选修复方向：临时或持久更换服务器 DNS 到可用解析器，然后无需关闭 Minecraft 服务端，等待新的登录请求验证；若要持久化需根据系统网络配置方式调整 `systemd-resolved` 或 netplan。

### 2026-07-07 运行时 DNS 修复

- 操作目标：修复正版验证阶段卡在“通信加密中”并提示身份验证服务器宕机的问题。
- 操作范围：修改云服务器运行时 DNS；未重启 Minecraft 服务端，未修改 `server.properties`，未关闭正版验证。
- 修改前 DNS：`223.5.5.5`
- 修改后运行时 DNS：
  - `1.1.1.1`
  - `8.8.8.8`
  - `119.29.29.29`
  - `114.114.114.114`
- 验证结果：
  - `sessionserver.mojang.com` 可解析到 `150.171.110.104`
  - `api.minecraftservices.com` 可解析到 `150.171.110.102`
  - `https://sessionserver.mojang.com/session/minecraft/hasJoined?username=Notch&serverId=0` 返回 HTTP `204`
  - `https://api.minecraftservices.com/minecraft/profile` 返回 HTTP `401`，说明服务可达但未携带认证令牌，属于预期结果
  - `hello-new-generation.service` 仍为 `active`
  - TCP/UDP `25565` 仍在监听
- 备注：本次是运行时 DNS 修复，通常不影响正在运行的 Minecraft 服务；如果云服务器重启或网络服务重载，DNS 可能恢复到原配置，届时需要做持久化 DNS 配置。

### 2026-07-08 只读 HTML 面板 Demo 部署

- 目标：在不影响 Minecraft 服务端运行的前提下，部署一个服务器本地只读 HTML 面板 demo，用于远程查看服务器信息。
- 操作范围：新增独立目录和独立 systemd 服务；未修改 Minecraft 服务端配置，未重启 Minecraft 服务端，未写入世界目录。
- 面板目录：`/data/minecraft/panel-demo`
- 面板服务：`mc-panel-demo.service`
- 运行用户：`minecraft`
- 监听地址：`0.0.0.0:18080`
- 认证方式：HTTP Basic Auth
  - 用户名：`admin`
  - 密码：仅保存在本地 `.local/servers.yaml`，不进入 Git 仓库。
- 本机验证结果：
  - `mc-panel-demo.service` 为 `active`
  - 面板监听 TCP `0.0.0.0:18080`
  - 面板 API 可返回服务端状态
  - `hello-new-generation.service` 仍为 `active`
  - Minecraft TCP/UDP `25565` 仍在监听
- 当前面板能力：
  - 查看 systemd 服务状态
  - 查看 Minecraft 25565 本机端口状态
  - 通过 Minecraft status ping 获取版本、MOTD、在线人数、最大人数
  - 读取部分 `server.properties`
  - 查看系统负载、内存、磁盘
  - 读取 `logs/latest.log` 的最近事件并做简单在线玩家推断
- 注意：这是 demo 版本，只做只读监控；HTTP Basic Auth 未加 HTTPS，远程公网访问时密码明文传输。后续正式版应放到 HTTPS 反代或 SSH/VPN/内网访问后面。
- 访问方式：优先尝试 `http://wzk.rainplay.cn:18080/`。如果雨云未开放直连端口，需要在雨云控制台把一个外部端口映射到服务器内部 `18080`。

### 2026-07-08 面板公网映射连通性检查

- 用户反馈：已将内网 `18080` 映射到 `wzk.rainplay.cn:56904`，但浏览器无法连接。
- 检查结果：
  - 服务器内部 `mc-panel-demo.service` 为 `active`
  - 面板监听 `0.0.0.0:18080`
  - 服务器内部未认证访问 `http://127.0.0.1:18080/` 返回 HTTP `401`
  - 服务器内部带认证访问 `http://127.0.0.1:18080/api/status` 返回 HTTP `200`
  - UFW 防火墙为 `inactive`
  - 从本地外部访问 `wzk.rainplay.cn:56904` TCP 连接成功
  - 从本地外部未认证访问 `http://wzk.rainplay.cn:56904/` 返回 HTTP `401 Unauthorized`
  - 从本地外部带认证访问 `http://wzk.rainplay.cn:56904/api/status` 返回 HTTP `200 OK`
- 判断：端口映射和面板服务正常。若浏览器仍无法打开，优先确认使用的是 `http://wzk.rainplay.cn:56904/`，不是 `https://`；并检查本地浏览器、代理或网络策略。

### 2026-07-08 面板中文化

- 操作目标：将只读 HTML 面板 demo 的界面文案改为中文。
- 操作范围：仅修改 `/data/minecraft/panel-demo/app.py` 并重启 `mc-panel-demo.service`；未修改 Minecraft 服务端配置，未重启 `hello-new-generation.service`。
- 处理内容：
  - 页面标题、说明、按钮、卡片标题改为中文。
  - 服务状态、端口状态、配置布尔值等常见字段改为中文显示。
  - 服务端配置项增加中文标签，并保留原始 key 便于后续开发对照。
  - 移除页面上的 Raw API 调试区块，减少英文技术噪音。
- 验证结果：
  - `mc-panel-demo.service` 为 `active`
  - `hello-new-generation.service` 仍为 `active`
- 访问建议：`http://wzk.rainplay.cn:56904/?v=cn2`

### 2026-07-08 Chunky 自动预生成与区块地图 Demo

- 目标：评估并初步实现“无人在线期间使用 Chunky 预生成区块”和网页可视化区块地图。
- 操作范围：仅修改并重启只读面板 `mc-panel-demo.service`；未修改 Minecraft 服务端配置，未重启 `hello-new-generation.service`，未执行任何 Chunky 指令，未写入世界存档。
- 依据：Chunky 官方 Wiki 中 `chunky start`、`chunky pause`、`chunky continue`、`chunky progress` 等指令可用于预生成任务管理。
- 当前限制：服务端 `enable-rcon=false`，面板无法向正在运行的 Minecraft 服务端发送命令。因此自动执行 Chunky 目前只做规划展示，不实际运行。
- 面板新增内容：
  - `Chunky 自动预生成（规划）`：显示 Chunky 模组是否存在、RCON 状态、当前在线人数、是否无人在线、建议指令。
  - `已生成区块地图（主世界）`：只读扫描 `world/region/*.mca` 的 region 文件头，统计并渲染已生成区块分布。
- 地图实现：
  - 不解压区块 NBT，不解析地形，不写入存档。
  - 读取 `.mca` 文件头中的 1024 个 chunk location entry，非零即认为该区块已生成。
  - 生成 BMP 位图供网页显示，亮色表示已生成区块密度，灰蓝轴线表示 X=0 / Z=0。
  - 地图结果缓存 120 秒，避免频繁扫描 region 文件。
  - 默认过滤绝对 region 坐标超过 512 的极远离群文件。
- 当前主世界统计：
  - Region 文件：`349 / 1254` 个非空主体 region 参与地图
  - 忽略极端 Region：`6`
  - 已生成区块：`276317`
  - 区块范围 X：`-4152 .. 4191`
  - 区块范围 Z：`-4472 .. 4291`
  - 方块范围 X：`-66432 .. 67071`
  - 方块范围 Z：`-71552 .. 68671`
  - 地图像素：`720 x 720`
  - 每像素约：X `11.59` 区块 / Z `12.17` 区块
- 后续真正自动化方案：
  - 需要开启 RCON（修改 `server.properties` 并重启 Minecraft 服务端生效）。
  - 面板通过 RCON 执行 `chunky continue` / `chunky pause` / `chunky progress`。
  - 建议策略：无人在线持续 10 分钟后继续预生成；有玩家进入立即暂停；限制每日运行时段和最大运行时长。
  - 不建议自动执行 `chunky trim`，该命令会删除区块，必须备份并人工确认。

### 2026-07-09 NeoForge TPS/MSPT 偏高只读诊断

- 操作范围：只读检查服务状态、系统负载、Java 进程、`latest.log`、JVM 参数和相关模组列表；未重启 Minecraft，未修改配置，未执行游戏内指令。
- 当前服务状态：
  - `hello-new-generation.service` 为 `active/running`
  - Java PID：`66040`
  - 面板服务 `mc-panel-demo.service` 仍在运行，但不是高负载来源
- 系统负载概况：
  - `load average` 约 `1.23, 0.91, 0.63`
  - `vmstat` 采样显示 CPU 大部分时间仍空闲，磁盘等待很低
  - Java 进程约占 `40%` CPU，主线程采样中短时约 `50%`
- 内存概况：
  - 服务 cgroup `MemoryCurrent` 约 `14.27GB`
  - 机器总内存约 `15GB`，可用内存约 `2.2GB`，swap 未使用
  - JVM 参数为 `-Xmx12G -Xms8G`
- 日志现象：
  - 当天 `latest.log` 中出现 `20` 次 `Can't keep up! Is the server overloaded?`
  - 多数卡顿出现在玩家登录、进服后短时间、移动/探索或区块加载附近
  - `AllTheLeaks` 每 10 分钟报告一次内存泄漏，主要对象包含：
    - `ServerPlayer (minecraft): 185-187`
    - `BlockTestLevel (supplementaries): 1`
  - `AllTheLeaks` 报告的内存差值在约 `+1.9GB` 到 `+3.4GB` 之间波动，最近约 `+2.78GB`
- 配置/模组线索：
  - `view-distance=10`
  - `simulation-distance=8`
  - `sync-chunk-writes=true`
  - 已安装性能相关模组：`spark`、`servercore`、`modernfix`、`ferritecore`、`lithium`、`c2me`、`noisium`
  - 重度玩法模组较多，包括大量 Create 系列、Sable / Valkyrien Skies 相关内容、Supplementaries 等
- 初步判断：
  - 当前不像是整台机器 CPU 被打满，更像是主服务器线程在玩家登录、区块加载、载具/机械/多维度保存，以及内存泄漏/GC 压力叠加时出现短时阻塞。
  - 内存余量偏紧和 `AllTheLeaks` 报告值得优先关注；长期运行后可能逐渐加重。
- 建议下一步：
  - 由 OP 玩家在卡顿明显时执行 Spark profiler，获得可定位到具体模组/方法的性能报告。
  - 后续可考虑降低 `view-distance` / `simulation-distance`，评估 `sync-chunk-writes=false`，并安排低峰定时重启或排查 `ServerPlayer` 泄漏来源。

### 2026-07-09 Spark 报告 nrbbJeIDzn 分析

- 数据来源：用户提供的本地文件 `E:\Downloads\nrbbJeIDzn.sparkprofile`，与公开链接下载文件 SHA256 一致。
- 报告时间：`2026-07-08 16:58:50 UTC` 到 `16:59:51 UTC`，约 60 秒；北京时间为 `2026-07-09 00:58:50` 到 `00:59:51`。
- 运行状态：NeoForge `21.1.233`，Minecraft `1.21.1`，采样时 1 名玩家在线。
- TPS/MSPT：
  - TPS：`20.000`，未出现持续掉 TPS。
  - 1 分钟 MSPT：平均 `26.84ms`，P95 `31.40ms`，最大 `60.81ms`。
  - 5 分钟 MSPT：平均 `29.27ms`，P95 `37.05ms`，最大 `804.59ms`。
  - 判断：不是持续崩 TPS，而是 tick 平均偏重，并伴随偶发尖峰。
- 系统状态：
  - Java 进程 CPU 约 `22%`，整机 CPU 约 `20%`，不是整机 CPU 打满。
  - 堆内存约 `7.89GB / 12GB`，已提交约 `10.36GB`。
  - 物理内存约 `13.46GB / 15.62GB`，内存余量偏紧，但本次 Spark 中 GC pause 不像主因。
- 主线程时间结构：
  - 60 秒采样中，约 `28.8s` 在 `waitUntilNextTick` / `waitForTasks` 等待，约 `30.0s` 在实际 `tickServer`。
  - 说明服务端仍能维持 20 TPS，但每 tick 工作量较重。
- 主要热点：
  - 实体 tick：`EntityTickList.forEach` 约 `18.31s`，占总采样 `30.5%`，约占实际 tick 工作的大头。
  - 方块实体 tick：`Level.tickBlockEntities` 约 `5.41s`，其中 Create `SmartBlockEntityTicker.tick` 约 `4.72s`。
  - Create 流体网络：`FluidTransportBehaviour.tick` / `PipeConnection.manageFlows` / `FluidNetwork.tick` / `FluidTank.fill`，热点最终指向 `create_connected` 的 `FluidVesselBlockEntity.onFluidStackChanged`，约 `2.13s`。
  - Sable：`ServerLevel.handler$elm000$sable$tickPlotContainer` 约 `2.89s`；`ServerChunkCache.handler$ell000$sable$hasChunk` 在 Create 流体更新路径中也出现，单点 self 约 `0.93s`。
  - 实体 AI 查询：Creeper `AvoidEntityGoal.canUse`、Touhou Little Maid `MaidTemptGoal.canUse`、动物 AI 等大量调用 `Level.getEntities` / `EntityTypeTest.tryCast`。
  - 自然刷怪：`NaturalSpawner.spawnForChunk` 约 `1.22s`，其中 PassableFoliage 的碰撞形状判断约 `0.52s`。
- 实体数量：总实体 `358`。较多实体包括 `minecraft:skeleton 42`、`minecraft:cow 31`、`minecraft:chicken 30`、`primal:shark 30`、`minecraft:item 27`、`minecraft:pig 24`、`minecraft:creeper 21`、`minecraft:horse 21`。
- 初步判断：当前卡顿热点主要不是硬件不足，而是玩家附近加载区域中的实体 AI、Create 流体/管道/容器网络、Sable 子世界/区块钩子共同叠加。
- 建议：
  - 优先检查玩家基地附近的 Create 流体管道、Fluid Vessel、泵、储罐、闭环管网，尽量拆分大流体网络或减少频繁变化的流体容器。
  - 清理/减少加载区实体，尤其是怪物、动物、掉落物和 `primal:shark` 密集区域；同时做刷怪/洞穴照明或降低模拟距离。
  - 检查 Sable/VS 相关船只、子世界、地块容器是否持续加载或频繁保存。
  - 后续可考虑把 `simulation-distance=8` 降到 `6`，必要时 `view-distance=10` 降到 `8`。
  - 内存泄漏仍需关注，但本次 Spark 报告里 GC 暂不像直接卡顿主因。

### 2026-07-09 Spark 报告 ZO7q39aXYr 分析

- 数据来源：`docs/ZO7q39aXYr.sparkprofile`。已解析出结构化摘要到 `docs/ZO7q39aXYr.analysis.json`。
- 报告时间：`2026-07-09 13:56:16 UTC` 到 `13:57:17 UTC`，约 60 秒；北京时间约 `2026-07-09 21:56:16` 到 `21:57:17`。
- 运行状态：NeoForge `21.1.233`，Minecraft `1.21.1`，采样时 2 名玩家在线。
- TPS/MSPT：
  - TPS：1 分钟 `20.00`，5 分钟 `19.74`，15 分钟 `19.87`。
  - 1 分钟 MSPT：平均 `31.95ms`，P95 `42.70ms`，最大 `74.09ms`。
  - 5 分钟 MSPT：平均 `36.47ms`，P95 `53.18ms`，最大 `1160.31ms`。
  - 判断：比上一份报告更重，已出现 5 分钟视角下的轻微 TPS 下滑和更大的 tick 尖峰。
- 系统状态：
  - Java 进程 1 分钟 CPU 约 `31.6%`，整机 CPU 约 `28.3%`，不是整机 CPU 被打满。
  - 堆内存约 `5.31GB / 12GB`，已提交约 `10.82GB`。
  - 物理内存约 `13.96GB / 15.62GB`，swap 未使用。
- 主线程时间结构：
  - 约 `36.3s / 60s` 在实际 `tickServer`，约 `22.2s / 60s` 在等待下一 tick 或等待任务。
  - 实际 tick 工作占比高于上一份，说明服务器余量变小。
- 性能占用成分：
  - 实体 tick：`EntityTickList.forEach` 约 `20.88s`，占总采样 `34.8%`，约占实际 tick 工作 `57.5%`。
  - 方块实体 / Create：`Level.tickBlockEntities` 约 `6.72s`，占总采样 `11.2%`，其中 Create `SmartBlockEntityTicker.tick` 约 `5.86s`，Create 流体 `FluidTransportBehaviour.tick` 约 `3.26s`。
  - 区块 tick：`ServerChunkCache.tick` 约 `3.75s`，其中自然刷怪 `NaturalSpawner.spawnForChunk` 约 `1.15s`。
  - Sable / VS：`sable$tickPlotContainer` 约 `3.37s`；Sable 物理 `SubLevelPhysicsSystem.tickPipelinePhysics` 约 `3.24s`；Rapier 物理 `RapierPhysicsPipeline.physicsTick` 约 `2.61s`。
- 实体情况：总实体 `484`，比上一份 `358` 明显增加。
  - `primal:shark`：`138`，上一份为 `30`，是最大变化点。
  - `minecraft:skeleton`：`51`
  - `minecraft:item`：`46`
  - `minecraft:chicken`：`28`
  - `minecraft:cow`：`27`
  - `minecraft:zombie`：`26`
  - `minecraft:creeper`：`22`
  - `primal:bear`：`16`
- 实体密集区：
  - 区块 `-24,-12` 有 `40` 个实体，主要是猪、鸡、羊、掉落物。
  - 区块 `-7,-1` 有 `21` 条 `primal:shark`。
  - 区块 `-8,-1` 有 `19` 条 `primal:shark`。
  - 区块 `-6,-7` 有 `16` 条 `primal:shark`。
- 初步判断：
  - 当前主要不是硬件瓶颈，而是玩家附近实体数量，尤其 `primal:shark` 爆量，叠加 Create 流体网络和 Sable 物理系统。
  - 和上一份相比，最明显恶化点是实体总数增加，尤其 `primal:shark` 从 `30` 到 `138`。
- 建议：
  - 优先处理 `primal:shark` 密集区，确认是否存在异常刷怪或水域刷怪配置过宽。
  - 清理区块 `-7,-1`、`-8,-1`、`-6,-7` 附近水域生物；也清理 `-24,-12` 附近动物和掉落物。
  - 检查 Sable/VS 船只或子世界物理实体是否持续运转。
  - 继续检查 Create 流体管线和 Fluid Vessel，尤其大网络、闭环管道和高频变动容器。

### 2026-07-09 客户端 Spark 报告 NqXCoixnvR 分析

- 数据来源：`docs/NqXCoixnvR.sparkprofile`。已解析结构化摘要到 `docs/NqXCoixnvR.analysis.json`。
- 报告性质：这是客户端 Spark 报告，不是服务端报告；适合判断客户端渲染线程、光影、Voxy、地图、Create/Flywheel 等带来的本地帧率压力。
- 报告时间：`2026-07-09 14:10:27 UTC` 到 `14:11:27 UTC`，约 60 秒；北京时间约 `2026-07-09 22:10:27` 到 `22:11:27`。
- 客户端基础状态：
  - NeoForge `21.1.233`，Minecraft `1.21.1`。
  - CPU：`AMD Ryzen 9 7940H w/ Radeon 780M Graphics`，16 线程。
  - Java 堆内存约 `4.23GB / 6GB`，未显示堆内存耗尽。
  - 物理内存约 `13.5GB / 15.22GB`，系统内存余量偏紧；swap/pagefile 记录约 `29.87GB`，需要关注后台程序和系统换页。
- 客户端 tick 状态：
  - Spark 中客户端侧 TPS 约 `20.00`，1 分钟 MSPT 平均约 `5.12ms`，P95 约 `6.74ms`。
  - 这说明本次样本的主要矛盾不在客户端逻辑 tick，而在渲染链路。
- Render thread 主要压力：
  - `GameRenderer.render` 约 `24.64s / 32.52s`，Render thread 的大部分时间都在渲染。
  - Iris 光影阴影：`IrisRenderingPipeline.renderShadows` / `ShadowRenderer.renderShadows` 约 `7.23s`，占 Render thread 约 `22.2%`，是最大单项热点。
  - 方块实体渲染：`SodiumWorldRenderer.renderBlockEntities` 约 `3.81s`，约 `11.7%`。
  - Create/Flywheel/Colorwheel 实例化渲染：`VisualizationManagerImpl.render`、`ClrwlInstancedDrawManager.renderSolid`、`submitDraws` 等约 `8.6% - 11.2%`。
  - Iris 批量实体/半透明渲染：`batchedentityrendering` 相关约 `6.2%`。
  - Xaero 地图：`xaero.map.events.ClientEvents.handleRenderTick` 约 `5.6%`，其中地图像素取色 `MapPixel.getPixelColours` 约 `4.3%`。
  - Voxy 渲染：`DefaultChunkRenderer...voxy...injectRender` 约 `716ms`，约 `2.2%`。在本样本里不是主瓶颈。
  - 粒子渲染：约 `1.3%`，不是主要问题。
- OpenGL / GPU 迹象：
  - `nglDrawElementsInstancedBaseVertex`、`nglDrawElementsBaseVertex`、`nglBufferData` 等 self time 明显，说明有一定 GPU/驱动调用和缓冲上传压力。
  - 这类耗时 Spark 只能看到 CPU 侧等待/调用，不能直接证明 GPU 满载；需要配合 Windows 任务管理器或显卡监控看 GPU 3D 占用。
- Voxy 线程解释：
  - `Chunk Render Task Executor (x10)` 显示大量时间在 Voxy semaphore acquire，但约 `99.8% - 99.9%` 是等待，不是实际 CPU 计算。
  - `Dedicated Voxy Worker` 也几乎都在等待，实际处理只有几十毫秒级。
  - 因此这份样本里 Voxy 存在，但不是优先怀疑对象。
- 客户端实体：
  - 客户端世界快照总实体约 `52`。
  - 较多的是 `create:super_glue 10`、`simulated:honey_glue 6`、`automobility:hitbox 4`、`minecraft:glow_squid 4`、`create:stationary_contraption 3` 等。
  - 没有看到服务端报告中的 `primal:shark` 大量出现在客户端样本里，因此这次客户端帧率压力不是由鲨鱼密集区直接触发。
- 初步判断：
  - 这次客户端卡顿/帧率压力主要来自光影阴影、Create/Flywheel/Colorwheel、方块实体渲染和地图渲染。
  - Voxy 的缓存/区块线程在这份报告中更多表现为等待，不像主要瓶颈。
- 建议排查顺序：
  - 优先关闭或降低 Iris 光影阴影质量、阴影距离、阴影分辨率，再重新采样对比。
  - 在 Create 机器/船只/子世界附近和远离这些结构的位置分别采样，确认 Flywheel/方块实体压力是否随场景变化。
  - 临时关闭 Xaero 小地图/世界地图显示，再观察帧率差异。
  - 使用任务管理器或 MSI Afterburner 等观察 GPU 3D、显存、系统内存和换页；当前物理内存余量偏紧，后台程序可能放大卡顿。
  - Voxy 可作为后续对照项测试，但优先级低于光影、Create/Flywheel 和地图。

### 2026-07-10 服务器运行状态检查

- 检查时间：北京时间约 `2026-07-10 22:14`，服务器时间 `2026-07-10 14:14 UTC`。
- 操作性质：只读检查，未重启、未停止、未修改服务器配置。
- Minecraft 服务：`hello-new-generation.service` 为 `active (running)`，自 `2026-07-02 18:07:15 UTC` 起已连续运行约 7 天。
- Java 进程：PID `66040`，启动参数仍为 NeoForge `21.1.233` / Minecraft `1.21.1`。
- 端口监听：
  - `25565` 正在由 Java 监听，Minecraft 服务端端口正常。
  - `18080` 正在由面板 Python 服务监听。
- Minecraft status ping：版本 `1.21.1`，协议 `767`，玩家 `0/20`。
- 面板服务：`mc-panel-demo.service` 为 `active (running)`，监听 `0.0.0.0:18080`。
- 系统状态：负载约 `0.08, 0.07, 0.02`，swap `0B / 1.4Gi`，没有明显系统级压力。
- 内存状态：系统总内存约 `15Gi`，可用约 `4.4Gi`；systemd 显示 MC 服务当前内存约 `11.5G`，峰值约 `15.0G`。
- 近期日志：看到 AllTheLeaks 持续报告 `ServerPlayer (minecraft): 203` 的疑似内存泄漏；同时有 Sable 定期保存子世界日志。最近多次 `ye_fan_233 lost connection: Disconnected`，但服务端本身仍在运行并可响应状态查询。

### 2026-07-13 分层备份系统首次部署与阶段性验收

- 目标：在不增加 Minecraft 模组的前提下，部署每小时本地热备份、连续无人在线 30 分钟后上传
  百度网盘、本地保留 3 份和云端最多 100 份的备份系统。
- 源码位置：仓库 `ops/backup/`；服务器部署副本位于 `/root/minecraft-backup-deploy`。
- 环境确认：Minecraft 服务目录为 `/data/minecraft/hello-new-generation`，运行用户和组均为
  `minecraft`，该用户 HOME 为 `/data/minecraft`，数据盘为 ext4。
- 非标准存储排查：未发现符号链接、外部数据库配置、MySQL/MariaDB/PostgreSQL/Redis 服务或相关
  监听端口。唯一命名为 `.db` 的文件是 FancyMenu 的 23 字节 ASCII 文本，不是数据库。
- 百度网盘：官方 `bdpan 3.8.3` 安装在 `/data/minecraft/.local/bin/bdpan`，由 `minecraft` 用户
  完成 OAuth 登录；网页端已在目标应用目录确认同名账号检查文件。未记录 Token 或账号秘密。
- RCON：安装器备份并修改 `server.properties`，配置副本为
  `/data/minecraft/hello-new-generation/server.properties.before-backup-20260712-183240`；密码文件为
  `root:root 0600`。维护窗口重启后，RCON `list` 测试成功。公网不应映射 `25575`。
- 本地热备份：通过 `minecraft-backup-now local` 完成。归档
  `hello-new-generation_2026-07-13_02-51-58_+0800.tar.zst` 大小为 `3,216,938,664` 字节；
  SHA-256 校验、归档完整读取、`server.properties` 和 `world/level.dat` 存在性均通过。
- 安全状态：归档及标记属于 `minecraft:minecraft` 且权限为 `0600`；备份后再次执行 `save-on`
  返回保存已开启，Minecraft 服务保持 `active`。
- 离线上传测试：服务器时间 `2026-07-12 18:56:03 UTC` 首次检查开始 30 分钟安静期；
  `18:58:55 UTC` 再次检查显示剩余 `1628` 秒，证明没有提前上传。
- 当前暂停点：百度网盘大文件上传、离线稳定快照、共享锁和玩家公屏目视确认尚未验收；两个 timer
  均未启用。本地约 3 GB 的归档及 `.sha256`、`.ready` 保留待处理。
- 继续测试注意：暂停期间 timer 未运行，无法证明玩家没有短暂上线。恢复验收时必须删除旧的
  `.offline-since` 和 `.offline-snapshot-for` 状态，从新的安静期开始，并让上传检查至少每 5 分钟
  运行一次。详细命令见 `ops/backup/README.md` 的“晚点继续首次上传验收”。
- 续测时发现原上传 timer 在系统启动很久以后临时 `start` 会立即消耗已经过期的 `OnBootSec=5min`
  触发点，表现为 timer 虽为 `active`，但 `NEXT` 为 `-`，不会继续检查。源码已改为
  `OnActiveSec=5min` 与 `OnUnitActiveSec=5min`，增加静态回归测试并重新部署；重新开始安静期后，
  `NEXT` 正确显示为 `2026-07-13 01:12:43 UTC`。该修正不涉及 Minecraft 重启。
- 新安静期在 `2026-07-13 01:38:22 UTC` 达标后开始离线快照，但 tar 严格检测到
  `world` 在读取期间变化并以失败退出；脚本删除了 partial、恢复 `save-on`，没有生成或上传可疑
  归档，Minecraft 保持 `active`。只读排查确认 `save-all flush` 在 `01:38:34` 保存多个世界数据，
  但 `world/level.dat` 到 `01:38:45` 才更新；`world/data/civil/meta.nbt` 又在 `01:42:14` 独立更新。
  曾试验在 flush 返回后等待 20 秒，同时保留严格变化检测。
- 加入 20 秒等待后的复测仍失败：第二次 flush 在 `01:48:09` 完成，但 `world/level.dat` 在
  `01:48:45` 再次更新，恰好比第一次的 `01:38:45` 晚 10 分钟。固定等待不能解决周期或异步写入，
  因而该实验改动已从源码撤回；服务器 timer 保持停止，脚本继续 fail-closed。若要求原子一致性，
  后续方案必须改为文件系统快照或经明确授权的停服冷备份。
- 后续对正式服 jar 进行只读反编译：Sable 1.2.2 的 `/sable paused` 只设置
  `SubLevelPhysicsSystem` 的 physics paused 状态，不暂停保存；Civillis 2.0.0 的管理员命令没有
  save/flush/pause，内部 `NbtStorage` 使用 `ColdIOQueue`，`close()` 会关闭存储并最多等待异步队列
  5 秒，不能在运行中安全调用后恢复。服主据此决定放弃热备份。
- 新源码策略：唯一 timer 每 5 分钟检查玩家，连续离线 30 分钟后正常停止
  `hello-new-generation.service`，制作 `_cold_` 冷备份，启动并等待 RCON 恢复后再上传；本地上限 3、
  云端硬上限 100。旧小时热备份单元由安装器停用并移除，旧不带 `_cold_` 的归档不会进入新上传队列。
- 新冷备份源码截至本记录仍只在 Windows 仓库中完成，尚未部署和执行正式服停启测试；服务器上的
  上传 timer 应继续保持 inactive，首次部署必须按 `ops/backup/HANDOFF.md` 在批准窗口人工观察完整
  停止、归档、恢复与上传流程。
- 回滚：备份调度可通过停用两个 timer 和服务撤回；RCON 配置如需撤回，必须在无玩家的维护窗口中
  使用上述 `server.properties` 副本谨慎恢复，具体步骤见 `ops/backup/HANDOFF.md`。

### 2026-07-13 冷备份首次成功与 bdpan 大文件上传失败

- 连续无人在线从 `05:44:12 UTC` 开始；`06:17:42 UTC` 最终检查通过后正常停止
  `hello-new-generation.service`。
- 冷归档从 `06:17:47` 到 `06:18:34`，生成
  `hello-new-generation_cold_2026-07-13_14-17-47_+0800.tar.zst`，大小 `3,216,465,220`
  字节，并生成 `.sha256` 与 `.ready`。
- Minecraft 自动启动并在 `06:20:09 UTC` 恢复 active 与 RCON；正式服停机窗口约 2 分 27 秒。
- bdpan 3.8.3 直接上传 3 GB 单文件到第 285 个分片时，在 `06:50:30 UTC` 报
  `context deadline exceeded`。网盘目录没有可见的同名完整文件，本地 `.ready` 保留。
- 失败后发现 timer 使用 `OnUnitActiveSec` 会在长任务退出时立即补跑，上传监视器也没有刷新玩家检查
  连续性。正式 timer 已人工停止，Minecraft 保持 active。
- 本地修正版改用 `OnUnitInactiveSec=5min`，上传期间每分钟刷新连续性；已有 `.ready` 时不再停服，
  失败后退避 60 分钟。云端改为 256 MiB 分卷逐个上传并按远端大小验证，最后上传
  `.parts.sha256` 作为完成标记，失败重试只补传缺失分卷。该修正版尚待重新部署与验收。
- 分卷实测中 `part-000` 成功，`part-001` 首次因 HTTP 504 / `ERR-f57192` 失败；人工重试后
  `part-001` 至 `part-006` 均成功并继续到 `part-007`，证明 256 MiB 分卷可以显著保留上传进度。
  `part-007` 又因 HTTP 504 / `ERR-9c840c` 失败，本地分卷与 `.ready` 均保留，Minecraft 全程 active。
- 再次人工重试时，父目录 `ls --json` 曾临时误判已存在的 `part-000` 为缺失并开始重复上传；操作员
  立即停止服务。随后只读查询确认父目录默认 `--limit 1000` 且仅有 8 项，`part-000` 至 `part-006`
  均存在并大小正确，因此不是分页遗漏，更可能是一次不完整或陈旧的目录响应。
- 本地后续修正不再用父目录列表判断单个分卷：上传前对完整远端路径做两次精确查询，只有明确
  `code=1 / 目录不存在` 才允许上传；名称、大小冲突、异常 JSON 或查询失败一律退避。上传后也用
  精确路径复核。云端上限计数改为按 `--start/--limit 1000` 分页遍历。该版本已部署并通过在线
  精确查询验收：已存在 `part-006` 返回 0，缺失 `part-007` 返回 1。
- v3 续传时正确跳过了远端 `part-000`，但查询 `part-001` 时百度接口返回“初始化根目录失败:
  context deadline exceeded”。脚本按不安全结果停止，没有重复上传。v4 将远端目录统计和精确路径
  查询改为每隔 10 秒最多重试 4 次；全部失败后才进入原有 60 分钟退避，明确缺失的双查询规则不变。
- v4 生产续传验收通过：远端目录统计前两次失败后自动重试成功；`part-000` 至 `part-008` 被精确
  验证并跳过，`part-009` 一次上传失败后保留本地文件，再次运行时从缺失分卷继续。最终
  `part-009` 至 `part-011`、归档 `.sha256` 和最后的 `.parts.sha256` 全部上传成功。
- `2026-07-13 10:06:52 UTC` 日志确认 `Cloud multipart upload verified (1/100)`；本地 `.ready`
  已改名为 `.uploaded`，临时分卷和本地分卷清单已删除，原始 3 GB 冷备份 SHA-256 复核为 `OK`。
  远端完成清单大小为 1620 字节，Minecraft 为 active，上传服务与 timer 均为 inactive。
- 首次全流程验收后正式启用唯一的 `minecraft-backup-upload.timer`；旧热备份单元不存在。第一次自动
  调度在 `2026-07-13 10:16:50 UTC` 触发，确认 0 人在线后从零开始 30 分钟安静期，未立即停服；
  Minecraft 为 active、任务服务为 inactive、timer 为 active，后续 5 分钟检查正常排程。
- 后续维护入口统一扩展为 `minecraft-backup-now`：提供状态、日志、本地 SHA-256 校验、云端只读列表、
  正常条件检查以及安全暂停/恢复。暂停不会强行中止正在运行的任务；恢复会先检查 Minecraft、RCON
  和 bdpan，并清除观察计时从零重新等待 30 分钟。README 与交接手册已补充成功判据和禁止操作。
- v5 维护入口已部署：11 项测试通过；生产调用 `status`、`logs`、自动分页 `cloud` 与 `resume` 成功。
  云端完成清单汇总为 1 份、大小 1620 字节。恢复 timer 时有 1 名玩家在线，命令先清除旧观察状态
  再安排下一次检查，避免沿用维护前的离线计时；Minecraft 保持 active。

### 2026-07-17 本地冷备份非覆盖式回档演练

- 目标：把正式服回档到
  `hello-new-generation_cold_2026-07-17_01-26-36_+0800.tar.zst`，同时完整保留回档前存档，接受约
  3 至 10 分钟维护窗口。
- 回档前先停用 `minecraft-backup-upload.timer`，确认没有正在运行的备份任务；目标归档先通过归档
  SHA-256、tar 流读取和关键文件检查，没有直接解压覆盖正式目录。
- 归档先解压到 `/data/minecraft` 下的独立暂存目录。切换前使用
  `/etc/minecraft-backup/rcon-password` 更新暂存副本的 `server.properties`，过程中没有回显或记录密码。
- 确认 RCON 返回 0 名玩家后，正常停止 `hello-new-generation.service`；服务完全停止后把原正式目录
  改名为 `/data/minecraft/hello-new-generation.before-rollback-20260717-114542`，再把暂存目录改名为
  `/data/minecraft/hello-new-generation`。因此回档没有覆盖或合并回档前存档。
- 已准备启动失败时自动把原目录换回的回滚保护，但本次没有触发。恢复后的服务为 active，日志出现
  `Done`，RCON 正常，FTB Quests 与 Sable 完成初始化；未发现 `level.dat` 损坏或世界启动崩溃。
- 启动日志仍有整合包既有的 Mixin、KubeJS、缺失 registry 项和配方解析告警；它们没有阻止服务端
  启动。玩家随后登录并实际游玩，确认本次恢复没有发现问题。
- 验收时正式目录与回档前安全目录均约 `5.5G`；`/data` 为 49G，使用 25G，可用 22G；本地备份目录
  约 13G。安全目录继续保留，没有在回档流程中清理。
- 验收期间自动 timer 保持 inactive/disabled。确认恢复正常后执行 `minecraft-backup-now resume`；
  状态显示 Minecraft active、timer enabled/active、备份任务 inactive，且有 2 名玩家在线，
  `Offline since`、`Last player check` 和上传退避状态均为空，证明观察期从零开始且没有误停服。
- 本次把“本地完整冷归档的非覆盖式恢复”从待验收改为已验证。尚未验证的灾难恢复边界是：从百度
  网盘下载全部分卷、按 `.parts.sha256` 校验和重组、再仅依赖云端副本完成相同切换。其他未主动触发
  边界仍包括本地第 4 份清理、云端 100 份硬上限和 bdpan Token 长期自动刷新。
