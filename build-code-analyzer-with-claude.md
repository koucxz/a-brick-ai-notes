# 用 Claude 写了个代码分析器

## 起因

前段时间接手了一个老项目，代码质量嘛...一言难尽。有个函数写了 400 多行，if 套 if 套了五六层，看得我头皮发麻。

然后我就想：**这玩意儿原理应该不复杂，要不自己写一个？**

说干就干。

## 我想要什么

先想清楚需求：

1. **能分析代码结构**：函数有多长、复杂度多高、参数有多少
2. **能检查代码规范**：命名是不是 snake_case、文件是不是太长
3. **能给出改进建议**：不只是报警，还要告诉我怎么改

前两个是传统静态分析工具干的事，第三个可以后面接 LLM。

## 整体架构

先画个图：

```
┌─────────────────────────────────────────────────────────┐
│                    CodeAnalyzer                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   Parser    │    │ RuleEngine  │    │  LLM Client │ │
│  │             │    │             │    │   (可选)    │ │
│  │ - Python    │───▶│ - 复杂度    │───▶│ - Ollama    │ │
│  │ - JS/TS     │    │ - 命名规范  │    │ - Prompt    │ │
│  │             │    │ - 结构检查  │    │             │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                  │                  │        │
│         ▼                  ▼                  ▼        │
│  ┌─────────────────────────────────────────────────┐   │
│  │              统一的分析结果                      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

三个核心模块：

1. **Parser**：把代码变成结构化数据
2. **RuleEngine**：按规则检查问题
3. **LLM Client**：调用大模型给建议（下一篇再讲）

这篇先搞定前两个。

## 用 Claude 来写代码

设计想清楚了，代码让 Claude 来写。

我的工作流是这样的：

1. **和Claude一起完成设计**：数据结构长什么样、接口怎么设计、模块怎么划分
2. **Claude 写实现**：具体的代码逻辑
3. **我来审查**：看看有没有问题，不满意就让它改

举个例子，我告诉 Claude：

> 我需要一个 Python 解析器，继承 ast.NodeVisitor，遍历语法树提取函数和类的信息。每个节点需要记录：名字、起止行号、复杂度、参数列表。复杂度就是数分支数量。

Claude 就能写出来。遇到细节问题再讨论，比如"怎么区分函数和方法"、"箭头函数的名字怎么拿"。

这种方式效率很高，断断续续几个晚上就把核心功能写完了。

## AST 解析

AST（抽象语法树）是代码分析的基础。简单说就是把代码变成一棵树，程序就能"看懂"代码了。

Python 自带 `ast` 模块：

```python
import ast
tree = ast.parse(code)
```

JS/TS 用 Tree-sitter，支持几十种语言，而且代码有语法错误也能解析。

### 复杂度计算

说白了就是数分支：if、for、while、try-except、and/or 这些都算。

```
基础 = 1
+ if/elif/for/while 各 +1
+ except 各 +1
+ and/or 各 +1
= 最终复杂度
```

一般超过 10 就该考虑拆分了。

### 区分函数和方法

Python 里函数和方法长得一样，都是 `def`。怎么区分？

看它在不在类里面。我的做法是维护一个 `current_class` 变量，进入类的时候设置，出来的时候清空。

### 统一输出格式

不管解析 Python 还是 JS，最后都输出同样的结构：

```python
@dataclass
class CodeNode:
    node_type: NodeType # 类型：函数/类/方法
    name: str           # 名字
    line_start: int     # 起始行
    line_end: int       # 结束行
    complexity: int     # 复杂度
    params: List[str]   # 参数列表
    decorators: List[str]  # 装饰器
    # ...
```

这样后面的规则引擎就不用关心语言差异了。

## 规则引擎

参考 ESLint 的设计，每个规则是一个类，实现 `check` 方法：

```python
class MaxComplexityRule(Rule):
    def check(self, parse_result):
        violations = []
        for func in parse_result.get_functions():
            if func.complexity > self.options['max']:
                violations.append(...)
        return violations
```

内置了 8 个规则：

| 规则 | 干嘛的 |
|------|--------|
| max-complexity | 复杂度别太高 |
| max-function-lines | 函数别太长 |
| max-params | 参数别太多 |
| function-naming | 函数用 snake_case |
| class-naming | 类用 PascalCase |
| max-file-lines | 文件别太长 |
| max-classes-per-file | 一个文件别塞太多类 |
| max-functions-per-file | 一个文件别塞太多函数 |

还支持配置文件，可以调整阈值、禁用规则：

```json
{
  "extends": "recommended",
  "rules": {
    "max-complexity": ["error", { "max": 8 }],
    "function-naming": "off"
  }
}
```

## 跑起来看看

```python
from a_brick_code_analyzer import PythonParser, RuleEngine

# 解析
parser = PythonParser()
result = parser.parse(code)

# 检查
engine = RuleEngine()
engine.use_preset('recommended')
lint_result = engine.lint(result)

for v in lint_result.violations:
    print(f"[{v.severity.name}] 第{v.line_start}行: {v.message}")
```

输出：

```
[WARNING] 第5行: 函数 'BadName' 应使用 snake_case 命名
[ERROR] 第5行: 函数 'BadName' 复杂度为 12，超过阈值 10
[WARNING] 第5行: 函数 'BadName' 参数过多 (7 > 5)
```

到这里，传统的静态分析部分就完成了。

## 小结

这篇主要讲了：

1. **为什么要做这个**：现有工具不顺手
2. **怎么用 Claude 写代码**：我定设计，它写实现
3. **AST 解析**：把代码变成结构化数据
4. **规则引擎**：按规则检查问题

但现在只能告诉你"哪里有问题"，不能告诉你"怎么改"。下一篇讲怎么接入 LLM，让它给出具体的修改建议。

---

**项目地址**：[GitHub - a-brick-code-analyzer](https://github.com/user/a-brick-code-analyzer)

有问题欢迎 Issue，觉得有用给个 Star~
