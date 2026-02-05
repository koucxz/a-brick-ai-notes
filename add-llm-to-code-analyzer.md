# 给代码分析器接上 LLM

## 上回说到

上一篇搞定了 AST 解析和规则引擎，能检查出代码的问题了。但用了几天发现一个痛点：

**规则只能告诉你"哪里有问题"，不能告诉你"怎么改"。**

比如它说"这个函数复杂度是 15，超过阈值了"。然后呢？具体怎么拆、拆成几个函数、每个函数干什么——还得自己想。

这不就是 LLM 擅长的事吗？

## 思路

我的想法很简单：

```
AST 分析结果 + 规则检查结果 + 源代码 → 组装成 Prompt → LLM → 具体建议
```

关键是怎么组装 Prompt。直接把代码丢给 LLM 让它审查，效果一般。但如果告诉它：

> 这个函数复杂度是 15，有 3 层嵌套，参数有 7 个。请给出重构建议。

效果就好多了。**结构化信息能帮 LLM 更好地理解问题。**

## 为什么选本地 LLM

这个项目本身就是个学习项目，用本地 LLM 正好一举两得：

1. **理解 LLM 工作原理**：亲手部署一遍，对模型加载、量化、推理流程有直观认识
2. **无限实验**：不用担心 API 费用，可以反复调试 Prompt、对比不同模型效果
3. **学习开源生态**：熟悉 Ollama、Hugging Face、GGUF 格式这些概念，以后用得上

Ollama 是目前最方便的本地部署方案：

```bash
# 安装
curl -fsSL https://ollama.com/install.sh | sh

# 拉模型
ollama pull deepseek-r1

# 跑起来
ollama serve
```

Windows 直接下载安装包，更简单。

### 模型怎么选

试了几个，说说感受：

| 模型 | 体验 |
|------|------|
| llama3.2:3b | 快，但有时候答非所问 |
| deepseek-r1 | 推理能力强，中文好，推荐 |
| qwen3 | 通用能力不错 |
| qwen3-coder | 代码场景最好，但要 24G 显存 |

显存有限就用 deepseek-r1，够用了。

## Prompt 设计

这是整个功能的核心。好的 Prompt 能让 LLM 给出有价值的建议，差的 Prompt 只会得到泛泛而谈。

### 模板结构

以代码审查为例：

```
你是一个专业的代码审查专家。请审查以下代码并提供改进建议。

## 代码信息
- 文件: {file_path}
- 语言: {language}
- 总行数: {total_lines}

## AST 分析结果
{ast_summary}

## 规则检查结果
{lint_results}

## 源代码
{code}

请从以下方面进行审查：
1. 代码质量: 可读性、可维护性、命名规范
2. 潜在问题: Bug 风险、边界情况、异常处理
3. 最佳实践: 是否遵循语言惯例和设计模式
4. 改进建议: 具体的优化方案

请给出具体的代码示例。
```

几个要点：

1. **给角色**：告诉 LLM 它是"代码审查专家"
2. **给上下文**：AST 分析结果、规则检查结果
3. **给结构**：明确要从哪几个方面回答
4. **要具体**：强调"给出具体的代码示例"

### 复杂度分析的 Prompt

这个场景比较有意思，专门针对"函数太复杂怎么拆"：

```
你是一个代码优化专家。请分析以下代码的复杂度问题。

## 复杂度较高的函数
- `process_data`: 复杂度 15 (行 10-85)
- `validate_input`: 复杂度 12 (行 90-130)

## 源代码
{code}

请分析：
1. 复杂度来源: 是什么导致了高复杂度
2. 影响评估: 这种复杂度会带来什么问题
3. 重构方案: 具体的简化步骤和代码示例
```

注意我把"复杂度较高的函数"单独列出来了，这样 LLM 就知道重点关注哪些函数。

## 整合：CodeAnalyzer

把 AST、规则引擎、LLM 串起来：

```
┌────────────────────────────────────────────────────────┐
│                    CodeAnalyzer                        │
│                                                        │
│   代码 ──▶ Parser ──▶ ParseResult                     │
│              │              │                          │
│              │              ▼                          │
│              │        RuleEngine ──▶ LintResult       │
│              │              │              │           │
│              ▼              ▼              ▼           │
│         ┌─────────────────────────────────────┐       │
│         │         PromptBuilder               │       │
│         │   组装: 代码 + AST + 规则结果       │       │
│         └─────────────────────────────────────┘       │
│                          │                             │
│                          ▼                             │
│                    OllamaClient                        │
│                          │                             │
│                          ▼                             │
│                    LLM 分析结果                        │
└────────────────────────────────────────────────────────┘
```

核心流程：

1. **解析代码**：用 Parser 提取结构信息
2. **规则检查**：用 RuleEngine 找出问题
3. **构建 Prompt**：把代码、AST 摘要、规则结果组装起来
4. **调用 LLM**：发给 Ollama，拿到建议

### 格式化 AST 摘要

不能把整个 AST 丢给 LLM，太长了。提取关键信息：

```
### 函数 (3 个)
- `process_data(items, config)` [复杂度: 15] (行 10-85)
- `validate_input(data)` [复杂度: 12] (行 90-130)
- `format_output(result)` [复杂度: 3] (行 135-150)

### 类 (1 个)
- `DataProcessor` (行 5-160)
```

这样 LLM 一眼就能看出哪些函数有问题。

### 格式化规则结果

```
共 4 个问题 (错误: 2, 警告: 2)
- [ERROR] 行 10: 函数 'process_data' 复杂度为 15，超过阈值 10
- [ERROR] 行 90: 函数 'validate_input' 复杂度为 12，超过阈值 10
- [WARNING] 行 10: 函数 'process_data' 参数过多 (7 > 5)
- [WARNING] 行 5: 类 'DataProcessor' 行数过多 (155 > 100)
```

## 实际效果

### 复杂度分析

输入一个嵌套很深的函数：

```python
def process_data(data, config, options=None):
    results = []
    for item in data:
        if item.get('type') == 'A':
            if item.get('status') == 'active':
                if config.get('mode') == 'strict':
                    value = item['value'] * 2
                else:
                    value = item['value']
                if options and options.get('transform'):
                    value = options['transform'](value)
                results.append(value)
            elif item.get('status') == 'pending':
                results.append(0)
        elif item.get('type') == 'B':
            if item.get('priority', 0) > 5:
                results.append(item['value'] * 3)
            else:
                results.append(item['value'])
    return results
```

LLM 的分析：

> **复杂度来源**
> 1. 嵌套条件过深：最深达到 4 层（type → status → mode → transform）
> 2. 多重分支：type 有 2 个分支，status 有 2 个分支，形成组合爆炸
>
> **重构建议**
> 建议按 type 拆分成独立函数：
> - `process_type_a(item, config, options)` - 处理 type A
> - `process_type_b(item)` - 处理 type B
> - `process_data(data, config, options)` - 主函数，负责分发

然后它还给出了重构后的代码示例。这就比单纯报个"复杂度超标"有用多了。

### 安全检查

输入一段有问题的代码：

```python
def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    cursor.execute(query)
    return cursor.fetchone()

def run_command(cmd):
    os.system(cmd)
```

LLM 的分析：

> **高风险**
>
> 1. **SQL 注入** (行 2)
>    - 问题：使用 f-string 拼接 SQL
>    - 修复：使用参数化查询
>    ```python
>    query = "SELECT * FROM users WHERE id = ?"
>    cursor.execute(query, (user_id,))
>    ```
>
> 2. **命令注入** (行 6)
>    - 问题：直接执行用户输入的命令
>    - 修复：使用 subprocess 并验证输入

这种场景 LLM 的价值很大，它能识别出安全问题并给出修复方案。

## 使用方式

封装了几个快捷方法，用起来很简单：

```python
from a_brick_code_analyzer import CodeAnalyzer

analyzer = CodeAnalyzer(model="deepseek-r1")

# 代码审查
result = analyzer.review(code)
print(result.content)

# 安全检查
result = analyzer.check_security(code)

# 复杂度分析
result = analyzer.analyze_complexity(code)

# 性能优化
result = analyzer.optimize_performance(code)

# 生成文档
result = analyzer.generate_docs(code)
```

也可以分析整个文件：

```python
result = analyzer.analyze_file("src/main.py")
```

## 一些坑

### LLM 会"幻觉"

有时候 LLM 会编造不存在的问题，或者给出不太靠谱的建议。所以它的输出只能作为参考，不能完全信任。

我的做法是：**AST 和规则检查的结果是准确的，LLM 的建议需要人工判断**。

### 本地模型能力有限

3B、7B 的模型能力确实不如 GPT-4 或 Claude。复杂场景可能分析不到位。

但对于常见的代码问题，够用了。而且本地跑不花钱、不泄露代码，这个优势很大。

### 速度

本地推理比较慢，7B 模型一次分析可能要 10-20 秒。

可以考虑：
- 只分析有问题的函数，不分析整个文件
- 用更小的模型做初筛，复杂问题再用大模型

## 总结

这篇讲了怎么把 LLM 接入代码分析器：

1. **Prompt 设计**：把 AST 分析结果和规则检查结果组装进 Prompt，让 LLM 有足够的上下文
2. **场景划分**：7 种分析场景，每种有专门的 Prompt 模板
3. **整合架构**：Parser → RuleEngine → PromptBuilder → LLM

核心思路还是那句话：**AST 负责"看清楚"，LLM 负责"想明白"**。

AST 能精确计算复杂度、提取结构信息，这是它的强项。LLM 能理解代码意图、给出改进建议，这是它的强项。两者结合，比单独用哪个都好。

---

**项目地址**：[GitHub - a-brick-code-analyzer](https://github.com/user/a-brick-code-analyzer)
