# Webhook-To

[![release](https://img.shields.io/github/v/release/Fromeverywhere/webhook-to?color=2f81f7)](https://github.com/Fromeverywhere/webhook-to/releases)
[![dockerhub](https://img.shields.io/docker/pulls/fromeverywhere/webhook-to?logo=docker&logoColor=white)](https://hub.docker.com/r/fromeverywhere/webhook-to)
[![license](https://img.shields.io/github/license/Fromeverywhere/webhook-to)](LICENSE)

**一站式处理 Webhook 通知。** 把任意应用、任意格式的上游通知收进来，做字段映射、模板渲染、条件过滤与路由，再统一转发到企业微信、飞书、Telegram、Bark 等十几种出站；也能完全脱离上游，按时间主动推送。

- **入站**：任意 JSON / form / multipart / XML / 纯文本，`POST /to/<规则名称>?key=<auth_key>`
- **处理**：字段映射 `extract` + 模板渲染 `render` + 条件过滤 / 分支路由 + HMAC 签名校验
- **出站**：企业微信群机器人、企业微信自建应用、wxpush、飞书、钉钉、Telegram、QQ（OneBot）、IYUU / Server酱、Bark、PushPlus、ntfy、Slack、Discord、任意 HTTP
- **定时通知**：独立页面管理，不依赖上游，按间隔或每天固定时间主动推送
- **反向回调**：在企业微信应用里点菜单 / 发一句话 → Webhook-To 调上游 API（MoviePilot 等）→ 把结果作为消息回给用户
- **管理**：内置 WebUI，「统计面板 / 入站规则 / 出站规则 / 定时通知 / 回调规则 / 通知监听 / 回调监听 / 日志 / 设置」九个页面，全流程可视化操作，无需手写 YAML

镜像标签 `1.1.1` / `1.0` / `latest`，`linux/amd64` + `linux/arm64` 同一份索引，GHCR 与 Docker Hub 内容一致。

**目录**：[安装](#安装) · [快速上手](#快速上手) · [核心概念](#核心概念) · [调用入口](#调用入口) · [预设库](#预设库) · [定时通知](#定时通知) · [回调验证](#回调验证企业微信-url-校验) · [回调规则](#回调规则反向控制) · [企微应用菜单](#企微应用自定义菜单) · [回调直通](#回调直通把企微回调原样转给上游) · [通知监听](#通知监听) · [回调监听](#回调监听) · [内置文档与新手指引](#内置文档与新手指引) · [安全](#安全) · [环境变量](#环境变量) · [数据目录](#数据目录) · [自己构建与发布](#自己构建与发布) · [常见问题](#常见问题)

---

## 安装

### Docker Compose（推荐）

```yaml
services:
  webhook-to:
    image: fromeverywhere/webhook-to:1.1.1
    container_name: webhook-to
    restart: unless-stopped
    ports:
      - "20010:8080"
    volumes:
      - ./data:/app/data
    environment:
      TZ: Asia/Shanghai
      RULES_DIR: data/rules/
    networks:
      - webhook-to

networks:
  webhook-to:
    name: webhook-to
    driver: bridge
```

```bash
mkdir -p data/rules
docker compose up -d
```

打开 `http://<host>:20010/` 即可。

- `./data:/app/data`：**一个目录装下全部状态**（入站规则在 `data/rules/`，其余是设置、凭据、出站规则、回调、监听、日志），备份 / 迁移只拷这一个文件夹。
- **离线导入**：`docker load < webhook-to-1.1.1.tar` 后把 `image:` 改成加载出来的镜像名即可。

### docker run

```bash
mkdir -p data/rules
docker run -d --name webhook-to --restart unless-stopped \
  -p 20010:8080 \
  -v "$PWD/data:/app/data" \
  -e TZ=Asia/Shanghai \
  -e RULES_DIR=data/rules/ \
  fromeverywhere/webhook-to:1.1.1
```

### 镜像与标签

| 标签 | 说明 |
| --- | --- |
| `fromeverywhere/webhook-to:1.1.1` | 精确版本（推荐固定用这个） |
| `fromeverywhere/webhook-to:1.0` / `latest` | 同一份镜像的移动别名 |
| `ghcr.io/fromeverywhere/webhook-to:1.1.1` | GHCR 同内容镜像，Docker Hub 不可达时换用 |

双架构（amd64 / arm64）由 CI 构建，GHCR 与 Docker Hub 同 digest。

---

## 快速上手

1. **登录**：首次部署默认账号 `admin` / 密码 `webhook-to`。登录后请到「设置 → 管理员账号」修改。
2. **建出站规则**（通知发到哪里）：「出站规则」页点预设磁贴（如企业微信群机器人），填上 key，「测试发送」确认能收到。
3. **建入站规则**（通知从哪里来）：「入站规则」页按上游项目挑一个预设（Emby / MoviePilot / Jellyfin …），改个规则名保存；或者从空白自定义。
4. **把调用地址交给上游**：入站规则卡片上的「复制地址」，粘到上游的 Webhook / 通知地址里，让上游发一条即可。

首页有**新手上手清单**全程引导（每一步自动打勾、带直达按钮）；右上角「❓ 操作指引」有 7 步聚光灯式分步向导，推荐先用预设、也可以自己写规则。

---

## 核心概念

| 概念 | 是什么 | 在哪配 |
| --- | --- | --- |
| **入站规则** | 一个上游入口：规则名决定地址 `/to/<名称>`，负责取字段（extract）和渲染消息（render） | 「入站规则」页 |
| **出站规则** | 消息发到哪：企微机器人 / 企微应用 / 飞书 / Telegram / 任意 HTTP 等 | 「出站规则」页 |
| **定时通知** | 无需上游，到点自动渲染并投递到选定的出站规则 | 「定时通知」页 |
| **回调规则** | 反向通路：用户在企微应用点菜单 / 发消息 → 调上游接口 → 结果回给用户 | 「回调规则」页 |
| **默认出站规则** | 入站规则没单独指定出站时统一走这条 | 「出站规则」页顶部 |
| **通知监听地址** | 每个地址对应一个上游应用，抓包记录独立成文件，可单独导出分析 | 「通知监听」页顶部 |
| **WorkBuddy 连接器** | 让 AI 在对话里查规则/日志/统计、发测试通知；两条接入路径（仅局域网 / 公网强制 2FA） | 「设置 → 连接 WorkBuddy」 |

典型数据流：

```
上游应用 ──POST /to/<规则名>?key=…──▶ 入站规则（extract + render）──▶ 出站规则 ──▶ 企微 / 飞书 / TG …
企微应用菜单/消息 ──/verify/<key>──▶ 回调规则 ──调上游 API──▶ 结果回复给用户
```

---

## 调用入口

| 入口 | 方法 | 说明 |
| --- | --- | --- |
| `/to/<规则名>?key=<auth_key>` | POST | 入站通知主入口；也支持 `X-Hook-Key` 请求头带 key |
| `/listen/<token>` | 任意 | 通知监听抓包，任何请求都会留痕并可导出；可在监听页新增多个地址，记录按地址分文件保存（`data/listener-<token>.jsonl`） |
| `/mcp` | POST | WorkBuddy 连接器（JSON-RPC/MCP），需接入令牌；默认仅局域网可连，公网路径强制 2FA 会话验证 |
| `/verify/<key>` | GET / POST | 回调验证端点（企业微信「接收消息服务器」URL 校验、加密消息） |
| `/callback/<规则名>?key=<auth_key>` | POST | 回调规则的明文入口（非企微通道用） |
| `/health` | GET | 健康检查，返回 `ok` |
| `/` | GET | 内置 WebUI（管理界面） |

鉴权说明：`key` 可以放 query（`?key=`）或 `X-Hook-Key` 请求头；入站规则保存时自动掩码，日志与 API 不落明文。

---

## 预设库

入站 / 出站规则都有**预设磁贴**：点一下就把字段映射、通知格式、样例报文全部填好，改个名字就能用。每个预设右上角「?」直达对应图文设置文档（参数从哪拿、上游后台一步步怎么点、常见错误）。

**入站预设**按项目分组：通用（任意上游，含图文卡片 / Markdown / 模板卡片多格式）→ Emby → Jellyfin → MoviePilot → Tautulli → Sonarr / Radarr → 下载工具（qBittorrent）→ 监控告警（Uptime Kuma / Alertmanager / Grafana）→ 网络与系统（Lucky DDNS / GitHub / Gitea）。

**出站预设**：企业微信群机器人、企业微信自建应用、飞书、钉钉、Telegram、QQ（OneBot）、IYUU、Server酱、wxpush 中转、Bark、PushPlus、ntfy、Slack、Discord、通用 HTTP。

> 预设在不断优化中，**不是必须的**：任何预设都可以随便改，也可以完全从空白自定义。不清楚上游真实报文长什么样？用「通知监听」抓一条照着填 Extract。

---

## 定时通知

「定时通知」页管理 `type: schedule` 的规则，**前提是先建好出站规则**，每条定时通知选一条出站规则来发：

- **按间隔**：`30m`（每半小时）、`1h`、`24h`，最小 `5s`
- **每天固定时间**：`09:00,18:30`（24 小时制，逗号分隔）

模板里可用内置变量：`{{._now}}`（`2026-01-01 12:00:00`）、`{{._date}}`、`{{._time}}`、`{{._source}}`。卡片上的「测试」随时手动发一条预览效果。

---

## 回调验证（企业微信 URL 校验）

「设置」页独立配置区块：

- **自定义 key**：默认 `verify-webhook-to`，地址就是 `GET/POST /verify/<key>`
- **自定义输出内容**：默认 `webhook-to ok`，可改任意文本与 Content-Type
- **企微加解密**：CorpID / 令牌 / EncodingAESKey 配好后，自动处理 `msg_signature` 校验（SHA1）与 AES-256-CBC 加密消息

企业微信「接收消息服务器配置」里：URL 填 `http://<host>:20010/verify/<key>`，Token 与 EncodingAESKey 填设置页对应值，点「保存」验证即可。

---

## 回调规则（反向控制）

与入站方向相反的通路：用户在企业微信应用里**点菜单**或**发一句话**，Webhook-To 按规则匹配（MsgType / Event / EventKey / 关键词 / AgentID）后去调上游接口（MoviePilot / qBittorrent / 自建脚本），再把结果作为消息回给用户。

- 通道：`wecom_app`（企微应用消息）/ `json`（明文 JSON 入口）/ `any`
- 动作模板内置 `{{._arg}}`（用户输入）、`{{._status}}`、`{{._ok}}`、`{{._body}}`、`{{._error}}`、`{{._r.xxx}}`（上游响应字段），支持 `${ENV_VAR}` 注入
- 异步执行 + MsgId 去重 + 5 秒内回复；失败会记录到「回调监听」
- 回调规则里含密钥的 URL / AuthKey 界面只回掩码，留空即保留原值

## 企微应用自定义菜单

「回调规则」页一键管理企微自建应用菜单（需要 CorpID / Secret / AgentID）：编辑 JSON → 发布 / 拉取 / 删除，access_token 自动缓存。菜单按钮的 EventKey 配到回调规则里就是一条反向指令。

## 回调直通（把企微回调原样转给上游）

回调规则的 `passthrough` 模式：把原始请求（method / query / body / Content-Type）**原样转发**到上游，响应原样回给调用方。适合把企微回调整体代理给 MoviePilot 这类自带处理逻辑的服务。URL / Query 支持模板与 `${ENV}`；企微通道返回空体即不回复用户。

---

## 通知监听

「监听任意通知」抓包页：`ANY /listen/<token>` 收下所有请求（任意方法、任意 Content-Type），拆出字段路径、格式化请求体，**可以直接照抄成入站规则的 Extract**。

- 历史保留 60 天；写过备注的记录永久保存
- 「导出为 TXT」：字段出现频次 + 格式化请求体 + 备注，整份可直接交给 AI 分析出规则
- 从监听记录一键「生成规则」跳入编辑器

## 回调监听

所有走到 `/verify/<key>` 与 `/callback/<name>` 的请求都留痕：URL 校验、企微解密后的字段、命中的规则、动作请求 / 响应、耗时与回给用户的消息。未命中与失败也记录，方便反推该补哪条菜单 EventKey 规则。同样支持「导出为 TXT」与备注。

---

## 内置文档与新手指引

- **文档中心** `http://<host>:20010/docs/index.html`：入站预设设置文档、出站预设设置文档、回调预设设置文档、《通知规则高级配置指南》（Extract 路径、模板函数、条件过滤、条件路由、签名校验、定时变量、监听、内部通知与失败补偿）
- **新手上手清单**：首页顶部五步清单（出站 → 测试 → 入站 → 交地址 → 改密码），完成自动打勾，每步带直达按钮，可随时「不再显示」
- **操作指引**：右上角 ❓ 或「设置 → 重看操作指引」，7 步聚光灯向导（出站用预设 / 自己写二选一，入站同理），ESC 或方向键操作
- **错误白话化**：连接失败、鉴权失败、模板语法错误等直接说人话并给出下一步建议

---

## 安全

「设置 → 安全」三个可选开关，全部默认关闭：

| 开关 | 作用 |
| --- | --- |
| **2FA 两步验证** | 勾选后生成密钥并展示二维码，用验证器（2FAS / Microsoft Authenticator 等）扫码，**输入 6 位动态码点「验证并启用」才正式开启**；登录时需再输动态码 |
| **登录验证码** | 登录前先看图输验证码（SVG，服务端校验） |
| **仅局域网** | 只允许局域网 IP 访问管理界面（公网入口自动 403） |

其他内置安全机制：登录失败 5 次锁 60 秒；密码只存 scrypt 哈希 + 随机盐；管理 API 支持会话 Cookie 或 Basic；写操作校验同源（CSRF）；auth_key / Secret 全链路掩码；日志自动过滤敏感串。

---

## 环境变量

只保留「启动前必须知道」的几项，其余全部在 WebUI「设置」页 / `settings.yaml` 里改完即生效：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `PORT` | `8080` | 监听端口 |
| `DATA_DIR` | `data` | 数据目录（设置、凭据、出站规则、监听、日志全在这里） |
| `RULES_DIR` | `rules/` | 入站规则目录；compose 范例指到 `data/rules/`，单卷备份更省心 |
| `UI_DIR` | `web/` | 内置 WebUI 静态资源目录（镜像内已带，通常不用改） |
| `TZ` | — | 时区，如 `Asia/Shanghai`（影响定时通知与日志时间） |

出站规则不走环境变量：在「出站规则」页配置，存 `data/targets.yaml`，保存即热加载。密钥推荐用 `${ENV_VAR}` 占位符 + 容器同名环境变量，避免明文落盘。

除上表部署路径类选项外，系统不再读取任何环境变量：账号密码、内部通知、企微 API 地址等全部在 WebUI 里配置。旧版本的 `DOWNSTREAM_URL` / `WECOM_API_BASE` 已于 1.0.2 移除。

## 数据目录

```
data/
├── rules/            # 入站规则 YAML（WebUI 保存与手改都会热加载）
├── settings.yaml     # 运行参数（超时 / 重试 / 队列 / 监听 Token / 内部通知 / 安全开关）
├── credentials.yaml  # 管理员账号（scrypt 哈希，不存明文）
├── targets.yaml      # 出站规则
├── callbacks/        # 回调规则
├── logs.jsonl        # 投递日志（轮转）
├── failed.jsonl      # 失败补偿队列
├── listener.jsonl    # 通知监听记录（60 天）
├── callback_monitor.jsonl  # 回调监听记录
└── ...
```

备份 / 迁移：整个 `data/` 拷走即可。

---

## 自己构建与发布

```bash
# 本地跑
go build -o webhook-to .
./webhook-to

# 全量测试（vet + 单测 + 构建 + e2e）
bash test.sh

# 无 Docker 环境手工打 docker-save 镜像 tar
python tools/build_image.py
```

发布链路（CI 已配置）：打 `v*` tag → GitHub Release 附 `webhook-to-linux-amd64` 二进制 → 构建多架构镜像推 GHCR（`1.1.1` / `1.0` / `latest`）→ 工作流同步 Docker Hub（digest 一致）。

---

## 常见问题

**上游发了通知但群里没收到？**
先看「日志」页的失败原因；再用「通知监听」抓一条确认上游真的发出来了、字段名和 Extract 对得上；最后到「出站规则」点「测试发送」确认出站链路通。

**改了规则 YAML 不生效？**
规则目录支持热加载，保存后 1~2 秒内生效；如果没生效，看日志是否有 YAML 解析错误（字段拼错会明确报出来）。

**入站规则的「出站规则」下拉是空的？**
先去「出站规则」页建一条；也可以不管下拉（留空 = 走「默认出站规则」）。

**密钥会泄露到日志里吗？**
不会。auth_key / Secret 全链路掩码，日志与 API 只回 `****`；想看明文点 WebUI 右上角眼睛（会话内生效）。

**能在公网暴露吗？**
可以，但务必：改默认密码 + 配 2FA + 每条入站规则设好 Auth Key + 需要时开启「仅局域网」。公网入口建议再套一层反向代理加 HTTPS。

**定时通知 / 回调怎么配？**
见「内置文档与新手指引」里的《通知规则高级配置指南》，或直接看 WebUI 弹窗顶部的折叠帮助。

---

## License

[MIT](LICENSE)
