---
name: seekdb-ai-functions
description: 在 OceanBase seekdb 中用 SQL 调用大模型——注册模型与 Endpoint（DBMS_AI_SERVICE），用 AI_PROMPT 组织提示词，用 AI_COMPLETE 生成文本、AI_EMBED 生成向量、AI_RERANK 重排文档。适用于“在数据库里直接让模型总结 / 分类 / 打标签 / 向量化 / 重排”这类任务，以及排查 AI 函数报错。
---

# seekdb AI 函数助手

seekdb 把大模型调用做成了 SQL 函数：数据不用导出，在 `SELECT` 里就能逐行调用模型。本 Skill 帮你完成三件事：

1. 正确注册模型和 Endpoint；
2. 按任务选对函数、写出 SQL；
3. 出错时按错误信息定位原因。

## 第 1 步：注册模型和 Endpoint（每个模型一次）

AI 函数通过**模型名**找到 Endpoint，再按 Endpoint 的 `provider` 选择请求协议。

```sql
-- type: completion / dense_embedding / rerank
CALL DBMS_AI_SERVICE.CREATE_AI_MODEL('my_llm', '{"type":"completion","model_name":"qwen-plus"}');
CALL DBMS_AI_SERVICE.CREATE_AI_MODEL_ENDPOINT('my_llm_ep', '{
  "ai_model_name": "my_llm",
  "url": "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions",
  "access_key": "<你的 API Key>",
  "provider": "openai",
  "max_concurrency": 8
}');
```

| 字段 | 说明 |
| --- | --- |
| `url` | 完整的接口地址，不是只填域名 |
| `provider` | `openai`、`aliyun-openai`、`aliyun-dashscope`、`deepseek`、`siliconflow`、`cohere`、`hunyuan-openai`、`ollama`，不区分大小写 |
| `access_key` | 本地 Ollama 可以留空，其他服务必填 |
| `request_model_name` | 可选：服务端模型名与 `model_name` 不同时填写 |
| `max_concurrency` | 可选：该 Endpoint 的最大并发请求数，JSON 整数 1–1024 |

常用的 Endpoint 写法：

| 服务 | provider | url 示例 |
| --- | --- | --- |
| OpenAI 兼容对话 | `openai` | `.../v1/chat/completions` |
| OpenAI 兼容向量 | `openai` | `.../v1/embeddings` |
| 硅基流动重排 | `siliconflow` | `https://api.siliconflow.cn/v1/rerank` |
| 本地 Ollama 生成 | `ollama` | `http://127.0.0.1:11434/api/generate` |
| 本地 Ollama 向量 | `ollama` | `http://127.0.0.1:11434/api/embed` |

修改和删除：

```sql
CALL DBMS_AI_SERVICE.ALTER_AI_MODEL_ENDPOINT('my_llm_ep', '{"max_concurrency": 4}');
CALL DBMS_AI_SERVICE.DROP_AI_MODEL_ENDPOINT('my_llm_ep');
CALL DBMS_AI_SERVICE.DROP_AI_MODEL('my_llm');
SELECT ENDPOINT_NAME, AI_MODEL_NAME, URL, PROVIDER, MAX_CONCURRENCY
FROM oceanbase.DBA_OB_AI_MODEL_ENDPOINTS;
```

创建、修改、删除分别需要 `CREATE AI MODEL`、`ALTER AI MODEL`、`DROP AI MODEL` 权限。

## 第 2 步：按任务选函数

| 任务 | 函数 | 返回值 |
| --- | --- | --- |
| 拼装提示词 | `AI_PROMPT(template, arg0, arg1, ...)` | JSON：`{"template": ..., "args": [...]}` |
| 生成文本（总结、分类、抽取） | `AI_COMPLETE(model, prompt [, config])` | LONGTEXT |
| 生成向量 | `AI_EMBED(model, text [, dimension])` | 向量的 JSON 数组文本 |
| 重排文档 | `AI_RERANK(model, query, documents [, text_field])` | 按相关度排序的 JSON 数组 |

### AI_PROMPT：模板 + 参数

- 模板中的 `{0}`、`{1}` 会被对应参数替换，同一个占位符可以出现多次。
- 参数可以是字符串、TEXT 列，也可以是 JSON：JSON 原样渲染成 JSON 文本；嵌套的 `AI_PROMPT(...)` 会先渲染成文字。
- `{}`、`{x}` 等不是占位符，原样保留。下标超出参数个数会报 `invalid placeholder index`。

```sql
SELECT AI_COMPLETE('my_llm',
         AI_PROMPT('把下面的评论分类为 正面/负面/中性，只输出类别：{0}', comment))
FROM reviews LIMIT 10;
```

### AI_COMPLETE：生成文本

第三个参数是 JSON 对象，原样合入请求，例如 `temperature`、`max_tokens`；传 NULL 等于不传。provider 为 Ollama 时会自动换算成 `options` 和 `num_predict`，并关闭流式输出。

```sql
SELECT id, AI_COMPLETE('my_llm', CONCAT('用一句话总结：', content), '{"temperature":0.2,"max_tokens":64}') AS summary
FROM docs WHERE created_at > NOW() - INTERVAL 1 DAY;
```

### AI_EMBED：生成向量

- 不传 `dimension` 时使用模型的默认维度；传入时必须是正整数，返回向量的维度必须与之相等。
- 输入为 NULL 时返回 NULL，不发请求。
- 批量执行时，同一批次的多行会合并成少量请求，结果按原行对应。

```sql
UPDATE docs SET emb = AI_EMBED('my_embed', content, 1024) WHERE emb IS NULL;
```

### AI_RERANK：重排

```sql
-- 字符串数组：返回 [{"index": 原位置, "relevance_score": 分数}, ...]，分数从高到低
SELECT AI_RERANK('my_rerank', '数据库事务', '["文档A","文档B","文档C"]', NULL);
-- 对象数组 + 文本字段：返回原对象，按分数排序，并附加 "_model_score"
SELECT AI_RERANK('my_rerank', '数据库事务', JSON_ARRAYAGG(JSON_OBJECT('id', id, 'text', title)), 'text')
FROM docs WHERE category = 'db';
```

分数相同的文档保持原来的先后顺序；文档较多时会自动分批，返回的 `index` 始终是在整个输入数组中的位置。

## 第 3 步：排错

| 报错 | 常见原因与处理 |
| --- | --- |
| `ai model not found` | 模型名写错，或者还没有 `CREATE_AI_MODEL` |
| `The param 'access_key' is empty or null` | 非 Ollama 服务必须填写 `access_key` |
| `The value of param 'max_concurrency' is invalid` | 必须是 1–1024 的 JSON 整数，不能写成字符串 `"8"` |
| `http status code: 401/403` | API Key 无效或没有权限 |
| `http status code: 429/5xx` | 服务限流或出错：客户端会退避重试，仍失败就降低并发、稍后重试 |
| `unexpected http message: ...` | 返回内容不符合协议：URL 是否指到了正确的接口（对话 / 向量 / 重排），provider 是否与服务匹配 |
| `result dimension is not equal to dimension` | 模型不支持请求的维度，换维度或去掉 `dimension` 参数 |
| `invalid placeholder index` | 模板里的 `{N}` 超出了参数个数 |
| 查询超时 | 数据量大时先用 `LIMIT` 试跑，或者调大 `ob_query_timeout` |

## 使用建议

- 先用 `LIMIT 5` 小批量验证提示词和输出格式，再处理全表。
- 需要结构化输出时，在提示词里明确要求只输出 JSON，并用 `JSON_VALID()` 检查。
- 生成的向量可以写入 `VECTOR(n)` 列，配合向量索引做检索，再用 `AI_RERANK` 精排。
- 不要把 API Key 写进业务表或日志里；Endpoint 只需要配置一次。

<!-- OB-S:tml8UoImQWiAxuiwHiRmKA -->
