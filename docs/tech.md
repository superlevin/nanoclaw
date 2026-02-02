# NanoClaw 技術文件

本文件說明 NanoClaw 的程式碼架構、執行流程與關鍵技術。

---

## 目錄

1. [系統概述](#系統概述)
2. [架構圖](#架構圖)
3. [核心元件](#核心元件)
4. [執行流程](#執行流程)
5. [關鍵技術](#關鍵技術)
6. [安全模型](#安全模型)
7. [資料流向](#資料流向)
8. [目錄結構](#目錄結構)

---

## 系統概述

NanoClaw 是一個輕量級的個人 Claude 助理，透過 WhatsApp 進行互動。系統設計遵循以下原則：

- **簡單易懂**：單一 Node.js 程序，少量原始碼檔案
- **容器隔離**：代理程式在 Linux 容器中執行，提供作業系統層級的隔離
- **單一使用者**：專為個人使用設計，非通用框架
- **AI 原生**：透過 Claude Code 進行設定與除錯

### 技術堆疊

| 技術 | 用途 |
|------|------|
| **Node.js 20+** | 主程式執行環境 |
| **TypeScript** | 程式語言 |
| **Claude Agent SDK** | AI 代理核心引擎 |
| **Apple Container / Docker** | 容器隔離執行 |
| **Baileys** | WhatsApp Web 連線 |
| **SQLite** | 訊息與任務儲存 |
| **Pino** | 日誌記錄 |

---

## 架構圖

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           使用者層 (User Layer)                               │
│                                                                             │
│    📱 WhatsApp                                                              │
│       │                                                                     │
│       ▼ (Baileys WebSocket)                                                 │
└───────┼─────────────────────────────────────────────────────────────────────┘
        │
┌───────▼─────────────────────────────────────────────────────────────────────┐
│                        主機程序 (Host Process)                               │
│                                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │  index.ts   │  │  db.ts       │  │ container-      │  │ task-         │  │
│  │  訊息路由   │──│  SQLite      │──│ runner.ts       │──│ scheduler.ts  │  │
│  │  IPC 監控   │  │  資料庫操作  │  │ 容器管理        │  │ 排程任務      │  │
│  └─────────────┘  └──────────────┘  └────────┬────────┘  └───────────────┘  │
│                                              │                              │
│  ┌─────────────────┐  ┌───────────────────────────────────────────────────┐ │
│  │ mount-          │  │                    config.ts                      │ │
│  │ security.ts     │  │  設定參數、路徑、觸發條件                         │ │
│  │ 掛載安全驗證    │  └───────────────────────────────────────────────────┘ │
│  └─────────────────┘                                                        │
└───────────────────────────────────────────────────────────────────────────┬─┘
                                                                            │
┌───────────────────────────────────────────────────────────────────────────▼─┐
│                      容器層 (Container Layer)                                │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Apple Container / Docker                           │   │
│  │  ┌────────────────┐  ┌────────────────────┐  ┌────────────────────┐  │   │
│  │  │  agent-runner  │  │  Claude Agent SDK  │  │  agent-browser     │  │   │
│  │  │  入口程式      │  │  AI 對話核心       │  │  瀏覽器自動化      │  │   │
│  │  └────────────────┘  └────────────────────┘  └────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────┐  │   │
│  │  │  ipc-mcp.ts - MCP 工具伺服器 (排程、訊息傳送、群組管理)       │  │   │
│  │  └────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  掛載目錄：                                                                 │
│  • /workspace/group    - 群組專屬目錄 (讀寫)                               │
│  • /workspace/global   - 全域記憶體 (唯讀，非主群組)                       │
│  • /workspace/ipc      - 程序間通訊目錄                                    │
│  • /workspace/extra/*  - 額外掛載目錄                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心元件

### 1. 主程式 (`src/index.ts`)

主程式負責整個系統的協調，包含以下功能：

```typescript
// 主要職責
- WhatsApp 連線管理 (makeWASocket)
- 訊息輪詢與路由
- IPC 監控與處理
- 排程器啟動
- 狀態管理
```

**關鍵函式：**

| 函式 | 說明 |
|------|------|
| `main()` | 系統入口點，初始化所有元件 |
| `connectWhatsApp()` | 建立 WhatsApp 連線 |
| `processMessage()` | 處理收到的訊息 |
| `runAgent()` | 啟動容器代理執行 |
| `startIpcWatcher()` | 監控 IPC 目錄 |
| `startMessageLoop()` | 主訊息處理迴圈 |

### 2. 容器執行器 (`src/container-runner.ts`)

負責在容器中執行 AI 代理：

```typescript
// 主要功能
- 建構掛載點配置
- 產生容器啟動參數
- 執行容器並處理輸出
- 管理任務與群組快照
```

**容器掛載邏輯：**

```typescript
// 主群組 (Main Group)
- /workspace/project  → 專案根目錄 (讀寫)
- /workspace/group    → 群組目錄 (讀寫)

// 其他群組
- /workspace/group    → 群組目錄 (讀寫)
- /workspace/global   → 全域目錄 (唯讀)
```

### 3. 資料庫模組 (`src/db.ts`)

使用 SQLite 儲存：

```sql
-- 聊天資訊
CREATE TABLE chats (
  jid TEXT PRIMARY KEY,
  name TEXT,
  last_message_time TEXT
);

-- 訊息內容
CREATE TABLE messages (
  id TEXT,
  chat_jid TEXT,
  sender TEXT,
  sender_name TEXT,
  content TEXT,
  timestamp TEXT,
  is_from_me INTEGER,
  PRIMARY KEY (id, chat_jid)
);

-- 排程任務
CREATE TABLE scheduled_tasks (
  id TEXT PRIMARY KEY,
  group_folder TEXT NOT NULL,
  chat_jid TEXT NOT NULL,
  prompt TEXT NOT NULL,
  schedule_type TEXT NOT NULL,  -- 'cron' | 'interval' | 'once'
  schedule_value TEXT NOT NULL,
  context_mode TEXT DEFAULT 'isolated',  -- 'group' | 'isolated'
  next_run TEXT,
  status TEXT DEFAULT 'active'
);
```

### 4. 排程器 (`src/task-scheduler.ts`)

管理定時任務的執行：

```typescript
// 排程類型
- cron: Cron 表達式 (如 "0 9 * * *" 每日9點)
- interval: 毫秒間隔 (如 "300000" 每5分鐘)
- once: 一次性執行 (ISO 8601 時間戳)

// 執行模式
- group: 使用群組對話上下文
- isolated: 獨立執行，無歷史記錄
```

### 5. 容器內代理 (`container/agent-runner/`)

容器內執行的代理程式：

```
container/agent-runner/
├── src/
│   ├── index.ts      # 代理入口，處理 stdin/stdout
│   └── ipc-mcp.ts    # MCP 工具伺服器
├── package.json
└── tsconfig.json
```

**IPC MCP 工具：**

| 工具 | 說明 |
|------|------|
| `send_message` | 發送訊息到 WhatsApp 群組 |
| `schedule_task` | 建立排程任務 |
| `list_tasks` | 列出所有任務 |
| `pause_task` | 暫停任務 |
| `resume_task` | 恢復任務 |
| `cancel_task` | 取消任務 |
| `register_group` | 註冊新群組 (僅主群組) |

---

## 執行流程

### 訊息處理流程

```
1. 使用者發送訊息
       │
       ▼
2. Baileys 接收 WebSocket 訊息
       │
       ▼
3. storeMessage() 儲存至 SQLite
       │
       ▼
4. startMessageLoop() 輪詢新訊息
       │
       ▼
5. 檢查觸發條件 (@Andy)
       │ 符合
       ▼
6. processMessage() 處理訊息
       │
       ▼
7. runAgent() 產生容器
       │
       ▼
8. container spawn → agent-runner
       │
       ▼
9. Claude Agent SDK 執行
       │
       ▼
10. 回應寫入 stdout (JSON)
       │
       ▼
11. sendMessage() 發送回覆
```

### 排程任務流程

```
1. startSchedulerLoop() 每分鐘執行
       │
       ▼
2. getDueTasks() 查詢到期任務
       │
       ▼
3. runTask() 為每個任務
       │
       ▼
4. runContainerAgent() 執行代理
       │
       ▼
5. 更新任務狀態與下次執行時間
       │
       ▼
6. logTaskRun() 記錄執行結果
```

### IPC 通訊流程

```
容器內 (agent-runner)              主機 (index.ts)
         │                              │
         │ 寫入 /workspace/ipc/tasks/   │
         │ ─────────────────────────▶   │
         │                              │ processTaskIpc()
         │                              │ 驗證權限
         │                              │ 執行操作
         │                              │
         │ 寫入 /workspace/ipc/messages/│
         │ ─────────────────────────▶   │
         │                              │ sendMessage()
         │                              │
```

---

## 關鍵技術

### 1. WhatsApp 連線 (Baileys)

使用 `@whiskeysockets/baileys` 實作 WhatsApp Web 連線：

```typescript
// 連線設定
const sock = makeWASocket({
  auth: { creds, keys },
  printQRInTerminal: false,
  logger,
  browser: ['NanoClaw', 'Chrome', '1.0.0']
});

// 事件監聽
sock.ev.on('messages.upsert', ({ messages }) => {
  // 處理新訊息
});

sock.ev.on('connection.update', (update) => {
  // 處理連線狀態
});
```

### 2. 容器隔離 (Apple Container / Docker)

容器提供作業系統層級的隔離：

```bash
# Apple Container 執行指令
container run -i --rm \
  -v /path/to/group:/workspace/group \
  --mount type=bind,source=/path/to/global,target=/workspace/global,readonly \
  nanoclaw-agent:latest
```

**容器 Dockerfile 重點：**

```dockerfile
FROM node:22-slim

# 安裝 Chromium 支援瀏覽器自動化
RUN apt-get update && apt-get install -y chromium ...

# 全域安裝工具
RUN npm install -g agent-browser @anthropic-ai/claude-code

# 非 root 使用者執行
USER node

# 入口點處理環境變數和 stdin
ENTRYPOINT ["/app/entrypoint.sh"]
```

### 3. Claude Agent SDK

代理程式使用 Claude Agent SDK 執行：

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({
  prompt,
  options: {
    cwd: '/workspace/group',
    resume: sessionId,  // 恢復對話
    allowedTools: [
      'Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep',
      'WebSearch', 'WebFetch',
      'mcp__nanoclaw__*'  // 自訂 MCP 工具
    ],
    permissionMode: 'bypassPermissions',
    mcpServers: { nanoclaw: ipcMcp }
  }
})) {
  // 處理訊息
}
```

### 4. MCP 工具伺服器

使用 SDK 建立自訂工具：

```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

createSdkMcpServer({
  name: 'nanoclaw',
  version: '1.0.0',
  tools: [
    tool(
      'send_message',
      '發送訊息到 WhatsApp 群組',
      { text: z.string().describe('訊息內容') },
      async (args) => {
        // 寫入 IPC 檔案
        writeIpcFile(MESSAGES_DIR, { type: 'message', text: args.text });
        return { content: [{ type: 'text', text: '訊息已排隊' }] };
      }
    )
  ]
});
```

### 5. 排程表達式 (Cron Parser)

使用 `cron-parser` 處理 Cron 表達式：

```typescript
import { CronExpressionParser } from 'cron-parser';

// 解析 Cron 表達式
const interval = CronExpressionParser.parse('0 9 * * *', { tz: TIMEZONE });
const nextRun = interval.next().toISOString();
```

### 6. 掛載安全驗證

防止容器存取敏感目錄：

```typescript
// 預設封鎖模式
const DEFAULT_BLOCKED_PATTERNS = [
  '.ssh', '.gnupg', '.aws', '.azure', '.gcloud',
  '.kube', '.docker', 'credentials', '.env',
  '.netrc', '.npmrc', 'id_rsa', 'id_ed25519',
  'private_key', '.secret'
];

// 驗證流程
1. 載入外部允許清單 (~/.config/nanoclaw/mount-allowlist.json)
2. 檢查是否符合封鎖模式
3. 驗證路徑是否在允許的根目錄下
4. 解析符號連結防止繞過
5. 根據權限設定唯讀或讀寫
```

---

## 安全模型

### 信任層級

| 實體 | 信任等級 | 說明 |
|------|----------|------|
| 主群組 | 受信任 | 私人自聊天，管理控制 |
| 非主群組 | 不受信任 | 其他使用者可能是惡意的 |
| 容器代理 | 沙盒隔離 | 受限的執行環境 |
| WhatsApp 訊息 | 使用者輸入 | 可能的提示注入攻擊 |

### 權限差異

| 功能 | 主群組 | 非主群組 |
|------|--------|----------|
| 專案根目錄存取 | ✓ | ✗ |
| 全域記憶體寫入 | ✓ | ✗ (唯讀) |
| 發送訊息到其他群組 | ✓ | ✗ |
| 排程其他群組任務 | ✓ | ✗ |
| 查看所有任務 | ✓ | 僅自己的 |
| 註冊新群組 | ✓ | ✗ |

### 容器隔離保護

```
┌─────────────────────────────────────────────────────┐
│                   不受信任區域                       │
│  WhatsApp 訊息 (可能的惡意內容)                     │
└────────────────────────┬────────────────────────────┘
                         │ 觸發檢查、輸入過濾
                         ▼
┌─────────────────────────────────────────────────────┐
│                   主機程序 (受信任)                  │
│  • 訊息路由                                         │
│  • IPC 授權驗證                                     │
│  • 掛載驗證 (外部允許清單)                          │
│  • 憑證過濾                                         │
└────────────────────────┬────────────────────────────┘
                         │ 僅掛載明確允許的目錄
                         ▼
┌─────────────────────────────────────────────────────┐
│               容器 (隔離/沙盒)                       │
│  • 代理執行                                         │
│  • Bash 指令 (容器內)                               │
│  • 檔案操作 (限於掛載目錄)                          │
│  • 無法修改安全設定                                 │
└─────────────────────────────────────────────────────┘
```

---

## 資料流向

### 檔案系統結構

```
nanoclaw/
├── src/                    # 主程式原始碼
│   ├── index.ts            # 主入口
│   ├── config.ts           # 設定
│   ├── db.ts               # 資料庫操作
│   ├── container-runner.ts # 容器管理
│   ├── task-scheduler.ts   # 排程器
│   ├── mount-security.ts   # 掛載安全
│   ├── types.ts            # 類型定義
│   └── utils.ts            # 工具函式
│
├── container/              # 容器相關
│   ├── Dockerfile          # 容器映像定義
│   ├── build.sh            # 建構腳本
│   └── agent-runner/       # 容器內程式
│       └── src/
│           ├── index.ts    # 代理入口
│           └── ipc-mcp.ts  # MCP 工具
│
├── groups/                 # 群組資料目錄
│   ├── main/               # 主群組
│   │   ├── CLAUDE.md       # 群組記憶
│   │   ├── conversations/  # 對話存檔
│   │   └── logs/           # 執行日誌
│   ├── global/             # 全域記憶
│   │   └── CLAUDE.md
│   └── {group-name}/       # 其他群組
│
├── data/                   # 運行時資料
│   ├── registered_groups.json  # 已註冊群組
│   ├── sessions.json           # 對話 session
│   ├── router_state.json       # 路由狀態
│   ├── sessions/{group}/.claude/ # 每群組 session
│   ├── ipc/{group}/            # IPC 目錄
│   │   ├── messages/           # 待發送訊息
│   │   ├── tasks/              # 任務操作
│   │   ├── current_tasks.json  # 任務快照
│   │   └── available_groups.json # 可用群組
│   └── env/                    # 憑證檔案
│
└── store/                  # 持久化儲存
    ├── auth/               # WhatsApp 認證
    └── messages.db         # SQLite 資料庫
```

### IPC 通訊檔案格式

**訊息 (`/workspace/ipc/messages/*.json`):**
```json
{
  "type": "message",
  "chatJid": "120363336345536173@g.us",
  "text": "這是回覆訊息",
  "groupFolder": "main",
  "timestamp": "2026-02-01T10:00:00.000Z"
}
```

**任務操作 (`/workspace/ipc/tasks/*.json`):**
```json
{
  "type": "schedule_task",
  "prompt": "每天早上發送天氣報告",
  "schedule_type": "cron",
  "schedule_value": "0 9 * * *",
  "context_mode": "isolated",
  "groupFolder": "main",
  "timestamp": "2026-02-01T10:00:00.000Z"
}
```

---

## 目錄結構

### 專案目錄說明

| 目錄/檔案 | 說明 |
|-----------|------|
| `src/` | TypeScript 原始碼 |
| `container/` | 容器建構檔案 |
| `groups/` | 群組資料和記憶 |
| `data/` | 運行時狀態資料 |
| `store/` | 持久化儲存 (認證、資料庫) |
| `docs/` | 文件 |
| `.claude/skills/` | Claude Code 技能定義 |
| `launchd/` | macOS 服務設定 |
| `config-examples/` | 設定範例 |

### 關鍵設定檔

**`data/registered_groups.json`:**
```json
{
  "120363336345536173@g.us": {
    "name": "Family Chat",
    "folder": "family-chat",
    "trigger": "@Andy",
    "added_at": "2026-01-15T10:30:00.000Z",
    "containerConfig": {
      "additionalMounts": [
        { "hostPath": "~/Documents", "containerPath": "docs", "readonly": true }
      ]
    }
  }
}
```

**`~/.config/nanoclaw/mount-allowlist.json`:**
```json
{
  "allowedRoots": [
    { "path": "~/projects", "allowReadWrite": true, "description": "開發專案" },
    { "path": "~/Documents", "allowReadWrite": false, "description": "文件 (唯讀)" }
  ],
  "blockedPatterns": ["password", "secret", "token"],
  "nonMainReadOnly": true
}
```

---

## 開發指令

```bash
# 開發模式 (熱重載)
npm run dev

# 編譯 TypeScript
npm run build

# 執行已編譯程式
npm start

# 類型檢查
npm run typecheck

# WhatsApp 認證
npm run auth

# 建構容器映像
./container/build.sh
```

### 服務管理 (macOS launchd)

```bash
# 載入服務
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist

# 卸載服務
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# 查看狀態
launchctl list | grep nanoclaw
```

---

## 相關文件

- [README.md](../README.md) - 專案介紹
- [REQUIREMENTS.md](./REQUIREMENTS.md) - 需求與設計決策
- [SECURITY.md](./SECURITY.md) - 安全模型詳細說明
- [CONTRIBUTING.md](../CONTRIBUTING.md) - 貢獻指南
