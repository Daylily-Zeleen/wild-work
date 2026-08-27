# TraeWork 网页版 API 逆向备忘

本文档记录 TraeWork (Solo) 公开端点及结构，按用途分类。

## 1. 核心模型接口

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/ide/v1/get_detail_param` | POST | 获取全量模型配置信息（返回 `config_info_list`，用于填充 `/v1/models`） |
| `/api/remote/v1/models` | GET | 拉取当前账号模型定价信息（速率、积分消耗） |
| `/agent/v3/llm_utils_chat` | POST | 核心聊天请求端点 |

## 2. 签到/积分接口

| 端点 | 方法 | 用途 |
|------|------|------|
| `/trae/api/v2/ug/checkin_credits/status` | GET | 获取签到状态 |
| `/trae/api/v2/ug/checkin_credits/claim` | POST | 签到领积分 |
| `/trae/api/v2/pay/web_user_ent_usage` | GET | 当前积分用量查询 |

## 3. Workspace 文件接口

### 3.1 HTTP 产物列表

| 端点 | 方法 | 用途 |
|------|------|------|
| `GET https://work.trae.cn/api/remote/v1/chat_sessions/{chatId}/output-artifact-metadata` | GET | 获取指定聊天会话的**输出产物元数据列表**（AI 在该会话生成的文件，每个文件包含 `id`/`name`/`path`/`created_at` 等） |

### 3.2 WebSocket 文件浏览

| 端点 | 类型 | 用途 |
|------|------|------|
| `wss://agent-sandbox-bj-d2-gw.trae.cn/explorer/{sessionId}` | WebSocket | 实时浏览 workspace 文件系统（列出目录/读取文件），需 session 鉴权 |
| `https://agent-sandbox-bj-d2-gw.trae.cn/explorer/{sessionId}/file?path=/{relativePath}` | HTTP GET | 直接下载指定路径文件内容 |

## 4. 说明

- 当前已记录已知文件相关端点
- 目前未发现列出**整个 workspace 所有文件**的公开 API
- 所有产物均存储在 Trae 官方域名（`trae.cn` / `trae-workspace.com` 子域名）下，无需鉴权即可直接下载（只要知道路径，token 在会话 context 中）