---
title: "基于 LangChain 的结构化输出实践"
date: 2023-12-25
tags: ["ai", "prompt", "llm", "structured-output", "langchaingo"]
categories: ["tech"]
---

在 AI 应用开发中，让大模型稳定地输出我们想要的格式是一个常见需求。本文介绍如何使用 LangChain 的 Golang 版本 [langchaingo](https://github.com/tmc/langchaingo) 实现 Prompt Template，并结合主流 LLM 的 JSON Mode 实现可靠的结构化输出。

## 什么是 Prompt Template

Prompt Template 是将提示词模板化的技术，通过占位符动态注入变量，让同一套提示词可以处理不同输入。类似于 Web 开发中的模板引擎。

<!--more-->

## langchaingo 简介

langchaingo 是 LangChain 的 Golang 实现，提供了与 LangChain Python 版类似的 API 设计，包括 Prompt Templates、Output Parsers、Chains 等核心组件。

```go
import "github.com/tmc/langchaingo/prompts"
```

## Prompt Template 基础

### 创建模板

langchaingo 支持多种模板格式，默认使用 Go 模板语法 `{{.variable}}`：

```go
template := prompts.NewPromptTemplate(
    "请分析以下文章，提取标题和作者。文章内容：{{.content}}",
    []string{"content"},
)

result, err := template.Format(map[string]any{
    "content": articleText,
})
```

### 多变量模板

```go
template, err := prompts.NewPromptTemplate(
    "将以下{{.input_language}}文本翻译为{{.output_language}}：{{.text}}",
    []string{"input_language", "output_language", "text"},
)

result, err := template.Format(map[string]any{
    "input_language":  "中文",
    "output_language": "英文",
    "text":            "你好世界",
})
```

### 支持多种模板格式

langchaingo 内置支持多种模板格式：

```go
import "github.com/tmc/langchaingo/prompts"

// F-string 格式（Python 风格）
fstringTemplate := prompts.PromptTemplate{
    Template:       "Hello {name}! Your score is {score}%.",
    InputVariables: []string{"name", "score"},
    TemplateFormat: prompts.TemplateFormatFString,
}

// Jinja2 格式
jinja2Template := prompts.PromptTemplate{
    Template:       "Hello {{ name }}! Your score is {{ score }}%.",
    InputVariables: []string{"name", "score"},
    TemplateFormat: prompts.TemplateFormatJinja2,
}
```

### Chat Prompt Template

用于构建多轮对话场景：

```go
chatTemplate := prompts.NewChatPromptTemplate([]prompts.MessageFormatter{
    prompts.NewSystemMessagePromptTemplate(
        "你是一个专业的{{.role}}，负责回答相关领域的问题。",
        []string{"role"},
    ),
    prompts.NewHumanMessagePromptTemplate(
        "{{.question}}",
        []string{"question"},
    ),
})

messages, err := chatTemplate.FormatMessages(map[string]any{
    "role":     "技术顾问",
    "question": "什么是 RAG？",
})
```

## 结构化输出实践

### 定义输出结构

使用 JSON Schema 描述期望的输出格式：

```go
articlePrompt := prompts.NewPromptTemplate(
    `请分析以下文章，提取结构化信息。
文章内容：{{.content}}

请严格按以下 JSON 格式输出，不要包含其他内容：
{
    "title": "文章标题",
    "author": "作者名",
    "published_at": "发布时间（YYYY-MM-DD格式）",
    "summary": "文章摘要（100字以内）",
    "tags": ["标签1", "标签2"]
}`,
    []string{"content"},
)
```

### 调用 LLM 并解析

langchaingo 与各种 LLM Provider 无缝集成：

```go
import (
    "github.com/tmc/langchaingo/llms"
    "github.com/tmc/langchaingo/llms/openai"
)

// 初始化 LLM
llm, err := openai.New()
if err != nil {
    log.Fatal(err)
}

// 渲染模板
rendered, err := articlePrompt.Format(map[string]any{
    "content": articleText,
})

// 调用 LLM
response, err := llms.GenerateFromSinglePrompt(context.Background(), llm, rendered,
    llms.WithTemperature(0.3),
)

// 解析 JSON 结果
var articleInfo struct {
    Title     string   `json:"title"`
    Author    string   `json:"author"`
    Published string   `json:"published_at"`
    Summary   string   `json:"summary"`
    Tags      []string `json:"tags"`
}
json.Unmarshal([]byte(response), &articleInfo)
```

## 完整示例：文章信息提取服务

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    "github.com/tmc/langchaingo/llms"
    "github.com/tmc/langchaingo/llms/openai"
    "github.com/tmc/langchaingo/prompts"
)

type ArticleInfo struct {
    Title     string   `json:"title"`
    Author    string   `json:"author"`
    Published string   `json:"published_at"`
    Summary   string   `json:"summary"`
    Tags      []string `json:"tags"`
}

func main() {
    // 初始化 LLM
    llm, err := openai.New()
    if err != nil {
        log.Fatal(err)
    }

    // 定义 Prompt 模板
    template, err := prompts.NewPromptTemplate(
        `用专业技术报告风格分析文章内容，提取结构化信息。
内容：{{.content}}

分析要求：
- 假设读者是行业专业人士
- 识别核心技术观点和创新点
- 指出实践意义和局限性
- 摘要控制在150-200字

输出JSON格式：
{
    "title": "文章标题",
    "author": "作者",
    "published_at": "发布日期（YYYY-MM-DD）",
    "summary": "专业分析摘要（150-200字）",
    "tags": ["标签1", "标签2"]
}`,
        []string{"content"},
    )
    if err != nil {
        log.Fatal(err)
    }

    // 提取文章信息
    article, err := ExtractArticle(llm, template,
        "这是一篇关于人工智能的技术文章，作者是张三，发布于2024年1月15日，主要讨论了大语言模型的应用。")
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("标题：%s\n", article.Title)
    fmt.Printf("作者：%s\n", article.Author)
    fmt.Printf("发布日期：%s\n", article.Published)
    fmt.Printf("摘要：%s\n", article.Summary)
    fmt.Printf("标签：%v\n", article.Tags)
}

func ExtractArticle(llm llms.LLM, template *prompts.PromptTemplate, content string) (*ArticleInfo, error) {
    // 渲染模板
    rendered, err := template.Format(map[string]any{
        "content": content,
    })
    if err != nil {
        return nil, fmt.Errorf("渲染模板失败: %w", err)
    }

    // 调用 LLM
    response, err := llms.GenerateFromSinglePrompt(context.Background(), llm, rendered,
        llms.WithTemperature(0.3),
    )
    if err != nil {
        return nil, fmt.Errorf("LLM 调用失败: %w", err)
    }

    // 解析结果
    var info ArticleInfo
    if err := json.Unmarshal([]byte(response), &info); err != nil {
        return nil, fmt.Errorf("解析 JSON 失败: %w", err)
    }
    return &info, nil
}
```

## 多 Provider 配置

langchaingo 支持多种 LLM Provider，便于在不同模型间切换。配置通过 YAML 文件管理：

```yaml
# config.yaml
providers:
  openai:
    provider: "openai"
    model: "gpt-4"
    base_url: "https://api.openai.com/v1"
    api_key: "${OPENAI_API_KEY}"

  deepseek:
    provider: "openai"  # 兼容 OpenAI 接口
    model: "deepseek-chat"
    base_url: "https://api.deepseek.com"
    api_key: "${DEEPSEEK_API_KEY}"

  ollama:
    provider: "ollama"
    model: "qwen2.5"
    base_url: "http://localhost:11434"
```

对应的 Go 配置加载逻辑：

```go
import (
    "github.com/tmc/langchaingo/llms"
    "github.com/tmc/langchaingo/llms/openai"
    "github.com/tmc/langchaingo/llms/ollama"
    "gopkg.in/yaml.v3"
)

type Config struct {
    Providers map[string]ProviderConfig `yaml:"providers"`
}

type ProviderConfig struct {
    Provider string         `yaml:"provider"`
    Model    string         `yaml:"model"`
    BaseURL  string         `yaml:"base_url"`
    APIKey   string         `yaml:"api_key"`
}

func NewLLM(cfg ProviderConfig) (llms.LLM, error) {
    switch cfg.Provider {
    case "openai":
        return openai.New(
            openai.WithModel(cfg.Model),
            openai.WithBaseURL(cfg.BaseURL),
        )
    case "ollama":
        return ollama.New(
            ollama.WithServerURL(cfg.BaseURL),
            ollama.WithModel(cfg.Model),
        )
    default:
        return nil, fmt.Errorf("unsupported provider: %s", cfg.Provider)
    }
}

// 使用示例
func main() {
    data, _ := os.ReadFile("config.yaml")
    var cfg Config
    yaml.Unmarshal(data, &cfg)

    llm, _ := NewLLM(cfg.Providers["openai"])
}
```

## 最佳实践

1. **明确输出格式**：在 Prompt 中给出完整的 JSON 示例，减少模型自由发挥空间
2. **使用 JSON Mode**：主流 LLM 的 JSON Mode 能显著提升结构化输出稳定性
3. **错误处理**：解析失败时保留原始响应，便于排查问题
4. **参数调优**：适当降低 temperature（0.3 左右）可让输出更确定性

## 总结

langchaingo 提供了与 LangChain Python 版一致的开发体验，Golang 的静态类型和并发优势使其非常适合构建高性能的 AI 服务层。通过 Prompt Template + JSON Mode 的组合，可以构建稳定可靠的结构化输出 pipeline。
