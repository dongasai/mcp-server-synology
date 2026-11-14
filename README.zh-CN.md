# mcp-server-synology

[English](./README.md) | 简体中文

基于群晖官方 DSM WebAPI 的 MCP 服务器，把一台群晖 NAS 暴露给 AI 客户端。

它用成套的专用工具覆盖 File Station、Download Station、Synology Photos、
Container Manager 和 DSM 系统管理；其余能力通过一个受管控的通用透传桥接访问
原始 DSM WebAPI。

## 为什么需要通用透传

DSM 对外暴露了数百个 API，群晖公开文档只覆盖其中一小部分。为每个 API 都造一
个工具既不现实也没意义，因此本项目分两层：

- **精选工具**：日常真正会用到的那批。它们会校验入参、轮询 DSM 的异步后台任
  务直到完成、把数字错误码翻译成可读句子，并强制执行路径白名单。
- **`list_dsm_apis` + `call_dsm_api`**：覆盖其余全部能力，例如 Surveillance
  Station、Hyper Backup、证书、用户与用户组、防火墙、Synology Drive、任务计
  划，以及未来 DSM 版本新增的任何 API。

透传不是提权。每次调用仍然以你配置的 DSM 账号身份执行，并受该账号权限约束。
它之所以单独设开关，只是因为它绕过了精选工具所施加的路径白名单和只读判定。

## 工具清单

默认策略（只读）下暴露 26 个工具；代码中共定义 41 个，写操作类工具在
`SYNOLOGY_READONLY=false` 时才出现。

### File Station —— 浏览与读取

| 工具 | 用途 |
| --- | --- |
| `list_shared_folders` | 列出该账号可访问的共享文件夹 |
| `list_files` | 列出目录，支持分页、排序和扩展名过滤 |
| `get_file_info` | 查看指定条目的大小、时间戳、属主和权限 |
| `search_files` | 按名称、扩展名、大小或修改时间做 DSM 索引搜索 |
| `read_file` | 在线读取文本文件，有大小上限且安全处理二进制 |
| `get_folder_size` | 递归统计目录大小与条目数 |
| `get_file_checksum` | 不下文件直接算 MD5 |

### File Station —— 写入、归档与分享

| 工具 | 用途 |
| --- | --- |
| `create_folder` | 创建目录，可选同时创建父级 |
| `write_file` | 上传文本或 base64 内容到文件 |
| `rename_file` | 原地重命名 |
| `move_files` / `copy_files` | 移动或复制，等待任务完成 |
| `delete_files` | 永久删除，单独加锁控制 |
| `compress_files` | 打包 ZIP 或 7z，可选加密 |
| `extract_archive` | 解压 ZIP、7z、RAR、TAR、GZ |
| `list_archive_contents` | 不解压直接查看压缩包内容 |
| `create_sharing_link` | 生成公开链接，可设有效期与密码 |
| `list_sharing_links` / `delete_sharing_link` | 审计并撤销公开链接 |

### Download Station

| 工具 | 用途 |
| --- | --- |
| `list_download_tasks` | 全部任务及进度、速度 |
| `get_download_task` | 指定任务的完整详情 |
| `create_download_task` | 排队 HTTP、FTP 或磁力下载 |
| `control_download_task` | 暂停、继续或移除任务 |
| `get_download_station_info` | 版本、吞吐与调度设置 |

### Synology Photos

| 工具 | 用途 |
| --- | --- |
| `list_photo_albums` | 个人空间或共享空间中的相册 |
| `list_photos` | 照片与视频，含 EXIF、位置和标签 |
| `list_photo_folders` | 浏览 Photos 目录树 |
| `search_photos` | 按主体、地点、人物或文件名搜索 |

### Container Manager

| 工具 | 用途 |
| --- | --- |
| `list_containers` | 容器列表，含状态、CPU 和内存 |
| `get_container_details` | 端口、挂载、环境变量、网络 |
| `control_container` | 启动、停止或重启 |
| `list_container_images` | 已存镜像及其体积 |

### 系统

| 工具 | 用途 |
| --- | --- |
| `get_system_info` | 型号、DSM 版本、序列号、运行时长、温度 |
| `get_resource_usage` | 实时 CPU、内存、磁盘 I/O 和网络 |
| `get_storage_info` | 存储空间、磁盘、剩余容量与 SMART 健康 |
| `list_installed_packages` | 已安装的 DSM 套件及其运行状态 |
| `list_shares_admin` | 共享文件夹配置与加密状态 |
| `list_active_connections` | 当前连接及其协议 |
| `control_system_power` | 重启或关机 |

### 通用透传

| 工具 | 用途 |
| --- | --- |
| `list_dsm_apis` | 发现这台 DSM 暴露的全部 API |
| `call_dsm_api` | 调用任意 DSM WebAPI 方法 |

## 安全模型

本服务器假定驱动它的是 AI 客户端，所以默认值一律取安全侧，每一项危险能力都
需要显式开启。

**第一层 —— DSM 账号。** 在 DSM 里建一个专用的非管理员用户，只授予它真正需
要的共享文件夹。本服务器做的任何事都不会超出该账号被允许的范围。绝不要指向
管理员账号。

**第二层 —— 策略开关。** 与 DSM 权限互相独立：

| 配置项 | 默认值 | 效果 |
| --- | --- | --- |
| `SYNOLOGY_READONLY` | `true` | 所有写操作工具被隐藏并拒绝 |
| `SYNOLOGY_ALLOW_DELETE` | `false` | 即使开了写入，删除仍被拒绝 |
| `SYNOLOGY_ALLOW_SYSTEM_CONTROL` | `false` | 不能重启、关机或控制容器 |
| `SYNOLOGY_ALLOW_GENERIC_API` | `false` | `call_dsm_api` 被隐藏 |
| `SYNOLOGY_ALLOWED_PATHS` | 空 | 把 File Station 限制到指定目录 |
| `SYNOLOGY_DENIED_PATHS` | 空 | 始终封禁的目录 |
| `SYNOLOGY_MAX_READ_BYTES` | `1048576` | 单次在线读取的大小上限 |

每次 File Station 调用前都会校验路径。含 `..` 的路径直接拒绝而不是规范化解
析 —— 因为解析后的路径仍可能借助 NAS 上的符号链接逃出白名单。

在当前策略下永远不可能成功的工具会从工具列表中隐藏，而不是先宣传再拒绝，避
免模型反复重试。

**第三层 —— 传输。** 设置 `SYNOLOGY_MCP_TOKEN` 后，每个 HTTP 客户端都必须
发送 `Authorization: Bearer <token>`。

### 建议的上线节奏

1. **只读。** 保持出厂默认，确认工具看到的内容符合预期。
2. **受限写入。** 设 `SYNOLOGY_READONLY=false`，并把 `SYNOLOGY_ALLOWED_PATHS`
   限定到一两个工作目录。
3. **运维操作。** 只有在你信任这套配置后，再开启删除、系统控制或通用透传。

## 配置

把 `.env.example` 复制成 `.env` 并填写。`SYNOLOGY_URL`、`SYNOLOGY_USER` 和
`SYNOLOGY_PASSWORD` 是必填项，其余都有安全的默认值。

如果 DSM 使用自签名证书，设 `SYNOLOGY_INSECURE_TLS=true`。该设置只作用于本
客户端的连接池，不会关掉整个进程的 TLS 校验。

## 运行方式

### 已发布镜像（GHCR）

每次推送到 `main`/`master` 都会构建多架构镜像并发布到 GitHub 容器镜像仓库：

```bash
docker pull ghcr.io/dongasai/mcp-server-synology:latest
docker run -d --name synology-mcp \
  --env-file .env \
  -e SYNOLOGY_MCP_TRANSPORT=http -e HOST=0.0.0.0 -e PORT=3000 \
  -p 127.0.0.1:3020:3000 \
  ghcr.io/dongasai/mcp-server-synology:latest
```

标签规则：`latest`（默认分支）、`<分支名>`、`sha-<提交号>`，推送 `v*` 标签时
额外生成 semver 标签。Pull Request 只构建镜像、不推送。镜像公开，拉取无需
`docker login`。

### Docker Compose

```bash
cp .env.example .env
$EDITOR .env
docker compose up -d --build
docker compose logs -f
```

容器默认只绑定 `127.0.0.1:3020`，因此只有本机反向代理能访问。用
`SYNOLOGY_HOST_BIND` 和 `SYNOLOGY_HOST_PORT` 覆盖。

### 本地 stdio

```bash
npm install
npm run build
SYNOLOGY_MCP_TRANSPORT=stdio node dist/index.js
```

## 访问位于家庭路由器后面的 NAS

托管的 MCP 服务器无法直接拨入家庭局域网。与其把 DSM 端口映射到公网，不如让
两个机器处在同一个私有覆盖网络上，把 `SYNOLOGY_URL` 指向该网络地址：

- **Tailscale** —— NAS 和运行本服务器的机器都装一遍，然后用 NAS 的 tailnet
  地址。最简单的方案，不用改路由器。
- **WireGuard** —— 如果你本来就有隧道，手工加一条。
- **frp / Cloudflare Tunnel** —— 由 NAS 主动发起的反向隧道。

只有本 MCP 服务器需要能连到 DSM，DSM 本身始终不暴露。

## 端点

| 路径 | 用途 |
| --- | --- |
| `POST /mcp` | MCP Streamable HTTP 端点 |
| `GET /healthz` | 存活检查：活跃会话数、暴露工具数、当前生效策略 |

`/healthz` 会回报当前生效的策略，因此不读容器环境变量也能审计运行时的安全姿
态。

## 值得了解的行为

- **会话续期。** DSM 会在空闲超时、重复登录和重启时让会话失效。客户端能识别
  错误码 106、107 和 119，重新认证一次并重放该调用，因此这不会表现为工具失败。
- **API 发现。** 接口路径和版本区间在运行时从 `SYNO.API.Info` 读取并缓存，所
  以同一个构建在 DSM 6 和 DSM 7 上都能用，也能应对在 CGI handler 之间搬家的
  API。
- **后台任务。** 复制、移动、删除、压缩、解压、目录大小和校验和都由 NAS 异步
  执行。每个工具会轮询到完成，而不是返回任务 ID；超时则报告部分进度。
- **错误消息。** DSM 会在 200 OK 的响应体里返回数字错误码，同一个数字在不同
  API 家族里含义不同。错误码会对照正确的码表翻译，并写成你能据此行动的描述。

## 参考

依据群晖公开的《File Station API》《Download Station API》和《DSM Login
Web API》文档，以及运行时通过 `SYNO.API.Info` 的发现。

## 许可证

MIT
