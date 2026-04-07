---
title: 'OpenClaw 内置引擎 + 硅基流动免费模型开启向量搜索'
date: 2026-03-31T19:30:00+08:00
draft: false
tags: ['openclaw', 'memory', 'embedding', 'ai']
categories: ['tech']
description: '折腾了 QMD、OpenViking 等方案后，最终用 OpenClaw 内置引擎 + 硅基流动免费的 bge-m3 模型，在低配服务器上实现了够用的向量混合搜索。'
---

上一篇[《OpenClaw 的 QMD 记忆引擎：从尝鲜到放弃》](/posts/openclaw-qmd-memory-engine/)里，我因为 2 核 4G 服务器跑不动 QMD 的 3 个本地 LLM 模型，切回了内置引擎。当时以为内置引擎只有关键词搜索——其实不是。

<!--more-->

## 内置引擎也支持向量搜索

翻了一遍 OpenClaw 官方文档才发现，内置的 SQLite 引擎本身就支持三种搜索模式：

| 模式 | 条件 | 说明 |
|------|------|------|
| **关键词搜索** | 默认 | SQLite FTS5 + BM25，纯文本匹配 |
| **向量搜索** | 配置 Embedding Provider | 基于语义相似度 |
| **混合搜索** | 配置 Embedding Provider | 关键词 + 向量，效果最好 |

关键区别在于：**有没有配 Embedding Provider**。没配就只有关键词搜索，配了就自动升级为混合搜索。

此外，内置引擎还有两个值得一提的能力：

- **sqlite-vec 加速**：OpenClaw 自带 `sqlite-vec` 扩展，向量查询直接在 SQLite 数据库内完成，不需要额外安装。如果加载失败会自动回退到进程内余弦相似度计算，功能不受影响。
- **CJK 支持**：通过三元组分词（trigram tokenization）支持中文、日文、韩文的全文检索，不需要额外配置分词器。

而 QMD 之所以慢，不是因为向量搜索本身慢，而是它在本地跑 3 个 LLM 做 `query expansion`、`embedding` 和 `reranking`。如果 embedding 走远端 API，开销就从"本地跑 1.7B 模型"变成了"一次 HTTP 请求"——毫秒级。

## 硅基流动的免费 Embedding 模型

硅基流动（SiliconFlow）提供了几个免费的 Embedding 模型，其中 `BAAI/bge-m3` 是多语言模型，中文效果好，而且完全免费。

更重要的是，硅基流动的 API **兼容 OpenAI 格式**。OpenClaw 内置引擎支持自定义 `remote.baseUrl`，所以可以直接把硅基流动当 OpenAI 用。

## 配置方法

在 `~/.openclaw/openclaw.json` 的 `agents.defaults` 里加上 `memorySearch`：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "BAAI/bge-m3",
        "remote": {
          "baseUrl": "https://api.siliconflow.cn/v1/",
          "apiKey": "你的硅基流动 API Key"
        }
      }
    }
  }
}
```

配置说明：

- `provider` 设为 `openai`，因为硅基流动兼容 OpenAI 的 `/v1/embeddings` 接口
- `model` 用 `BAAI/bge-m3`，1024 维，多语言，免费
- `remote.baseUrl` 指向硅基流动的 API 地址

保存后重启 OpenClaw，然后执行 `openclaw memory index --force` 强制重建索引。完成后用 `openclaw memory status` 确认状态：

```
$ openclaw memory status

Memory Search (main)
Provider: openai (requested: openai)
Model: BAAI/bge-m3
Sources: memory
Indexed: 20/20 files · 23 chunks
Dirty: no
Store: ~/.openclaw/memory/main.sqlite
Workspace: ~/.openclaw/workspace
By source:
  memory · 20/20 files · 23 chunks
Vector: ready
Vector dims: 1024
Vector path: ~/.nvm/versions/node/v22.22.0/lib/node_modules/openclaw/node_modules/sqlite-vec-linux-x64/vec0.so
FTS: ready
Embedding cache: enabled (23 entries)
Batch: disabled (failures 0/2)
```

几个关键指标：

- **Provider: openai** — 通过 OpenAI 兼容接口连接硅基流动
- **Vector: ready / Vector dims: 1024** — bge-m3 生成的 1024 维向量已就绪
- **Vector path: ...sqlite-vec-linux-x64/vec0.so** — sqlite-vec 加速已启用
- **FTS: ready** — 关键词全文搜索就绪
- **Indexed: 20/20 files · 23 chunks** — 所有记忆文件已索引

之后新增或修改记忆文件，OpenClaw 会自动监听变化并重新索引，不需要手动操作。

## 验证向量搜索是否生效

可以用 `openclaw memory search` 直接测试：

```bash
$ openclaw memory search "相册部署架构"

0.427 memory/photo-project.md:1-28
# 相册项目
- **现部署:** 腾讯云 Lighthouse + Nginx + PM2 + Next.js
...
```

搜索词"相册部署架构"在记忆文件里并不完整存在——文件里写的是"项目"、"架构"、"部署"分散在不同位置。纯关键词搜索不会命中，但向量搜索通过语义匹配精准找到了 `photo-project.md`。这就是混合搜索的价值。

## 三种方案终极对比

OpenClaw 的记忆搜索我前后折腾了三种方案，放在一起比较：

| | 内置引擎 + 硅基流动 | QMD | OpenViking |
|---|---|---|---|
| 搜索方式 | FTS5 + 向量混合 | BM25 + 向量 + rerank | 向量 + VLM 摘要 |
| Embedding | 远端 bge-m3 API | 本地 300M 模型 | 远端 API |
| 搜索延迟 | 毫秒级 | 1-2 分钟（2核CPU） | 毫秒级 |
| 中文效果 | ✅ bge-m3 多语言 | ⚠️ 偏英文 | ✅ doubao 中文好 |
| 费用 | 🆓 免费 | 🆓 免费（吃 CPU） | 💰 持续计费 |
| 硬件要求 | 无 | 高（需 GPU 或强 CPU） | 无 |
| 部署复杂度 | 改一段 JSON | 装 QMD | Python 服务 + systemd |
| 供应商锁定 | 无 | 无 | 有（数据在向量库） |

QMD 功能最全（有 `query expansion` 和 `reranking`），但在低配机器上跑不动；OpenViking 搜索质量最高，但要持续花钱且有供应商锁定；内置引擎 + 硅基流动是折中方案——没有 `reranking`，但免费、快、中文好、零部署成本。

对于 20-30 个记忆文件的个人使用场景，混合搜索已经绰绰有余。

## 总结

兜了一圈，最终的记忆搜索方案是：

- **存储**：内置 SQLite（Markdown 文件 → FTS5 索引 + 向量索引）
- **Embedding**：硅基流动 bge-m3（免费、中文好、远端计算）
- **搜索**：混合模式（关键词 + 向量）

不需要装额外软件，不吃本地 CPU，不花钱，中文效果还比 QMD 默认的英文模型好。对于跑在低配云服务器上的 OpenClaw 来说，这大概是最务实的方案了。
