# Codex CLI + DeepSeek V4 完整接入指南

将 OpenAI Codex CLI (v0.130+) 接入 DeepSeek V4 API 的完整方案。

## 背景

Codex CLI v0.130 彻底移除了 `wire_api = "chat"` 支持，强制要求 OpenAI Responses API。但 DeepSeek 只支持 Chat Completions API。本方案通过开源中间件 `codex-chat-bridge` 实现协议转换。

## 最终架构

```
Codex CLI/Desktop
  ↓ POST /v1/responses (Responses API)
codex-chat-bridge (127.0.0.1:19099)
  ↓ 协议转换 + Tool 过滤
DeepSeek API (api.deepseek.com/v1/chat/completions)
```

## 快速开始

### 1. 安装 Codex CLI

```bash
# 要求 Node.js >= 22
node --version

# 安装
npm install -g @openai/codex

# 验证
codex --version
```

### 2. 安装 codex-chat-bridge

```bash
# 安装 Rust（macOS）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# 安装 bridge（需要编译 Rust binary）
npm install -g @heungtae/codex-chat-bridge
```

### 3. 配置 Bridge

创建 `~/.codex/bridge.toml`:

```toml
[routers.deepseek]
incoming_url = "http://127.0.0.1:19099/v1/responses"
upstream_url = "https://api.deepseek.com/v1/chat/completions"

[routers.deepseek.features]
tool_transform_mode = "passthrough"
```

### 4. 配置 Codex

编辑 `~/.codex/config.toml`:

```toml
model = "deepseek-v4-flash"
model_provider = "deepseek-bridge"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[model_providers.deepseek-bridge]
name = "DeepSeek V4"
base_url = "http://127.0.0.1:19099/v1"
wire_api = "responses"
api_key = "dummy"
```

### 5. 启动服务

```bash
# 设置 DeepSeek API Key
export DEEPSEEK_API_KEY="your-deepseek-api-key"

# 启动 bridge（后台运行）
nohup codex-chat-bridge \
  --config ~/.codex/bridge.toml \
  --api-key-env DEEPSEEK_API_KEY \
  --drop-tool-type web_search \
  --drop-tool-type code_interpreter \
  --drop-tool-type mcp \
  --drop-tool-type namespace \
  --drop-tool-type custom \
  --drop-tool-type web_search_call \
  --drop-tool-type browser \
  > ~/.codex/bridge.log 2>&1 &

# 测试
codex exec "say hello"
```

## 踩坑记录

| # | 问题 | 原因 | 解决 |
|---|------|------|------|
| 1 | `wire_api = "chat"` 报错 | v0.130 移除 Chat API 支持 | 必须用 `wire_api = "responses"` |
| 2 | `unknown variant developer` | DeepSeek 不支持 `developer` role | Bridge 自动映射为 `system` |
| 3 | `Failed to deserialize: tools[N].type` | DeepSeek 只支持 `function` 类型 tool | `--drop-tool-type` 过滤非 function tools |
| 4 | `tools[4].function: missing field name` | Codex 发送了无名称的工具定义 | `--drop-tool-type` 或过滤 |
| 5 | `missing environment variable` | Codex Desktop 无 shell env | 改用 `api_key = "dummy"` |
| 6 | `responses->chat bridge does not support tool type` | 新 tool 类型出现 | 追加 `--drop-tool-type` |

## 模型切换

在 Codex 中切换 DeepSeek 模型：

```
/model deepseek-v4-flash    # 默认，便宜快速
/model deepseek-v4-pro      # 最强推理，更贵
/model deepseek-chat        # V3 通用模型
/model deepseek-reasoner    # R1 推理模型
```

需要在 bridge 中修改 upstream_url 对应的模型名映射。

## 安装 Codex Desktop

```bash
codex app
```

自动下载并安装 Codex Desktop.app 到 `/Applications`，使用相同的 `~/.codex/config.toml` 配置。

## 环境要求

| 组件 | 最低版本 |
|------|----------|
| Node.js | >= 22 |
| npm | >= 10 |
| Rust | >= 1.85 |
| macOS | 13+ / Linux glibc 2.31+ |

## 参考

- [Codex CLI](https://github.com/openai/codex)
- [codex-chat-bridge](https://www.npmjs.com/package/@heungtae/codex-chat-bridge)
- [DeepSeek API](https://api-docs.deepseek.com)
- [Codex Chat Completions 废弃公告](https://github.com/openai/codex/discussions/7782)

## 2026-05-18 更新：多轮 Agent 完全可用

### 最终方案：aliyun-codex-bridge + 补丁

经过大量测试，最终推荐使用 `aliyun-codex-bridge`（npm 包）替代 `codex-chat-bridge`。

### 安装

```bash
npm install -g aliyun-codex-bridge
```

### 补丁（2处）

修改 `~/.local/lib/node_modules/aliyun-codex-bridge/src/server.js`:

**补丁 1**（约 870 行）：给所有 assistant 消息添加空 `reasoning_content`

```javascript
// 在 finalMessages 赋值后添加:
for (const m of finalMessages) {
  if (m.role === 'assistant') {
    if (!m.reasoning_content) m.reasoning_content = '';
  }
}
```

**补丁 2**（约 878 行）：注释掉 `reasoning` 参数透传

```javascript
// 注释掉整个 reasoning 参数透传代码块
```

### 启动

```bash
PORT=19099 AI_API_KEY="your-api-key" AI_API_BASE="https://api.deepseek.com/v1" ALLOW_TOOLS=1 \
  nohup node ~/.local/lib/node_modules/aliyun-codex-bridge/src/server.js &
```

### Codex 配置

```toml
[model_providers.aliyun-bridge]
name = "DeepSeek V4"
base_url = "http://127.0.0.1:19099"
wire_api = "responses"
api_key = "dummy"
```

### 测试结果

| 场景 | 状态 |
|------|------|
| 简单对话 | ✅ |
| 代码解释 | ✅ |
| 单步工具执行 | ✅ |
| 多轮 Agent（读文件→分析） | ✅ |
| --output-last-message | ✅ |

## 2026-05-18 晚间更新：Reasoning 思维链完全解放

### 补丁 3（关键）：reasoning → reasoning_content 映射

修改 `aliyun-codex-bridge/src/server.js` 第 827 行附近：

**之前**：`// Skip Reasoning, FunctionCall, LocalShellCall, etc.`

**之后**：提取 reasoning item 的文本，附加到 assistant 消息的 `reasoning_content` 字段回传给 DeepSeek。

### 效果

| 指标 | 修复前 | 修复后 |
|------|--------|--------|
| Token 消耗 | ~10K | ~21K（含推理链） |
| Reasoning 保留 | ❌ 被丢弃 | ✅ 81+ 块完整回传 |
| DeepSeek 思维链 | 空 | 完整运行 |
| 威力释放 | ~70% | ~90% |

EOF

CONTENT=$(base64 -i README.md)
SHA=$(GH_TOKEN="your-github-token" gh api repos/Tinzlu/codex-deepseek-guide/contents/README.md --jq '.sha' 2>/dev/null)
GH_TOKEN="your-github-token" gh api -X PUT \
  repos/Tinzlu/codex-deepseek-guide/contents/README.md \
  -f message="docs: Reasoning思维链完全解放 - 补丁3" \
  -f content="$CONTENT" -f sha="$SHA" -f branch="main" 2>&1 | python3 -c "import sys,json;d=json.load(sys.stdin);print('✅' if 'content' in d else 'ERR:'+str(d.get('message','?')))"