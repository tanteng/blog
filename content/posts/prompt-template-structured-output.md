---
title: "Prompt Template 与结构化输出实战"
date: 2023-12-25
tags: ["AI", "Prompt", "DeepSeek", "结构化输出"]
categories: ["tech"]
---

在 AI 应用开发中，如何让大模型稳定地输出我们想要的格式？本文介绍如何使用 Golang 实现 Prompt Template，并结合 DeepSeek 的 JSON Mode 实现可靠的结构化输出。

## 什么是 Prompt Template

Prompt Template 是将提示词模板化的技术，通过占位符动态注入变量，让同一套提示词可以处理不同输入。类似于 Web 开发中的模板引擎。

## DeepSeek 的 JSON Mode

DeepSeek API 支持 JSON Mode，通过设置 `response_format` 参数为 `json_object`，可以强制模型输出有效的 JSON 字符串。

```go
request := &ChatCompletionRequest{
    Messages: []Message{
        {Role: "user", Content: "请解析以下JSON..."},
    },
    ResponseFormat: &ResponseFormat{
        Type: "json_object",
    },
}
```

## Golang 实战：可配置多模型的 Prompt Template 系统

### 1. 定义配置结构

```go
type Config struct {
    Providers map[string]ProviderConfig `json:"providers"`
}

type ProviderConfig struct {
    APIKey  string `json:"api_key"`
    BaseURL string `json:"base_url"`
    Model   string `json:"model"`
}
```

### 2. 定义 Prompt 模板

```go
type PromptTemplate struct {
    Template   string            `json:"template"`
    InputKeys  []string          `json:"input_keys"`
    OutputType string            `json:"output_type"`
}
```

### 3. 实现渲染函数

```go
func RenderTemplate(template string, data map[string]interface{}) string {
    re := regexp.MustCompile(`\{\{(\.?[\w.]+)\}\}`)
    return re.ReplaceAllStringFunc(template, func(match string) string {
        key := strings.Trim(match, "{}")
        if strings.HasPrefix(key, ".") {
            key = key[1:]
        }
        keys := strings.Split(key, ".")
        value := interface{}(data)
        for _, k := range keys {
            if m, ok := value.(map[string]interface{}); ok {
                value = m[k]
            } else {
                return match
            }
        }
        if value == nil {
            return match
        }
        return fmt.Sprintf("%v", value)
    })
}
```

### 4. 结构化输出示例

定义一个提取文章信息的 Prompt：

```go
articlePrompt := PromptTemplate{
    Template: `请分析以下文章，提取标题、作者、发布时间和摘要。
文章内容：{{.content}}
请以JSON格式输出：
{
    "title": "文章标题",
    "author": "作者名", 
    "published_at": "发布时间",
    "summary": "摘要内容"
}`,
    InputKeys:  []string{"content"},
    OutputType: "json",
}
```

调用并解析结果：

```go
result := RenderTemplate(articlePrompt.Template, map[string]interface{}{
    "content": articleText,
})

response := CallLLM(provider, articlePrompt.Template, data)
var parsed ArticleInfo
json.Unmarshal([]byte(response), &parsed)
```

## 完整示例：文章信息提取服务

```go
type ArticleInfo struct {
    Title      string   `json:"title"`
    Author     string   `json:"author"`
    Published  string   `json:"published_at"`
    Summary    string   `json:"summary"`
    Tags       []string `json:"tags"`
}

func ExtractArticle(content string, provider string) (*ArticleInfo, error) {
    prompt := `分析以下文章内容，提取结构化信息。
内容：{{.content}}

输出JSON格式：
{
    "title": "标题",
    "author": "作者",
    "published_at": "发布日期",
    "summary": "摘要",
    "tags": ["标签1", "标签2"]
}`

    rendered := RenderTemplate(prompt, map[string]interface{}{
        "content": content,
    })

    resp, err := CallLLM(provider, rendered)
    if err != nil {
        return nil, err
    }

    var info ArticleInfo
    if err := json.Unmarshal([]byte(resp), &info); err != nil {
        return nil, err
    }
    return &info, nil
}
```

## 多模型配置示例

```json
{
    "providers": {
        "deepseek": {
            "api_key": "sk-xxx",
            "base_url": "https://api.deepseek.com",
            "model": "deepseek-chat"
        },
        "openai": {
            "api_key": "sk-xxx", 
            "base_url": "https://api.openai.com",
            "model": "gpt-4"
        }
    }
}
```

## 最佳实践

1. **明确输出格式**：在 Prompt 中给出完整的 JSON 示例
2. **使用 JSON Mode**：启用 API 的结构化输出支持
3. **错误处理**：解析失败时保留原始响应便于调试
4. **模板验证**：确保必填变量都已注入

## 总结

通过 Prompt Template + JSON Mode 的组合，我们可以构建稳定可靠的 AI 应用。Golang 的静态类型和并发优势使其非常适合构建高性能的 AI 服务层。

---
