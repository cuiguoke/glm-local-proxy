本地 LLM 代理服务：将**标准 OpenAI Chat Completions API** 格式转换为**智谱 AI 国内版 API**（open.bigmodel.cn）能接受的格式。

## 为什么需要这个代理

智谱 AI 国内版 API 声称兼容 OpenAI API，但实际上只实现了 2023 年初的简化版本，与现代 OpenAI 客户端（如 rig-core）生成的请求格式存在多处不兼容：

| 问题 | OpenAI 标准格式 | GLM 国内版要求 |
|------|----------------|---------------|
| 消息内容 | `[{"type":"text","text":"..."}]` 数组 | 必须是字符串 `"..."` |
| Tool Schema | 支持 `anyOf`/`oneOf`/`allOf` | 不支持，会返回 1210 错误 |
| 类型定义 | `"type": ["string", "null"]` | 必须是单个字符串 `"type": "string"` |
| 额外属性 | `"additionalProperties": false` | 不支持 |

这些不兼容导致使用 rig-core 的应用（如 IronClaw）在调用 GLM 国内版 API 时持续报错：

```
{"error":{"code":"1210","message":"Invalid API parameter, please check the documentation."}}
```

`glm-local-proxy` 作为中间层，在请求转发前自动修复这些格式问题，无需修改上层应用代码。

## 编译

```bash
# 在 ironclaw workspace 根目录下编译
cargo build -p glm-local-proxy --release

# 二进制文件位于
./target/release/glm-local-proxy
```

## 使用

### 1. 启动 LLM 代理

```bash
ZAI_API_KEY=your_api_key ./target/release/glm-local-proxy
```

代理默认监听 `127.0.0.1:4000`。

### 2. 配置 IronClaw

在 `.env` 文件中：

```bash
LLM_BACKEND="openai_compatible"
LLM_BASE_URL="http://127.0.0.1:4000/v1"
LLM_API_KEY="any"
LLM_MODEL="glm-4"
```

### 3. 验证

```bash
curl http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"glm-4","messages":[{"role":"user","content":"你好"}]}'
```

## 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ZAI_API_KEY` | 智谱 AI API Key（必填） | - |
| `GLM_BASE_URL` | GLM API 地址 | `https://open.bigmodel.cn/api/paas/v4` |
| `PROXY_ADDR` | 代理监听地址 | `127.0.0.1:4000` |
| `RUST_LOG` | 日志级别 | `glm_local_proxy=debug` |

API Key 在 [智谱 AI 开放平台](https://open.bigmodel.cn/usercenter/apikeys) 获取。

## 代理处理的转换

**消息内容扁平化**

```json
// 输入（rig-core 生成）
{"role": "user", "content": [{"type": "text", "text": "你好"}]}

// 输出（GLM 接受）
{"role": "user", "content": "你好"}
```

**Tool Schema 清洗**

```json
// 输入（rig-core 生成）
{
  "type": "object",
  "properties": {
    "query": {"type": ["string", "null"]},
    "body": {"description": "请求体"}
  },
  "additionalProperties": false,
  "anyOf": [{"required": ["query"]}]
}

// 输出（GLM 接受）
{
  "type": "object",
  "properties": {
    "query": {"type": "string"},
    "body": {"type": "string", "description": "请求体"}
  }
}
```
