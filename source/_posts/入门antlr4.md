---
title: 入门 ANTLR 4
date: 2025-02-11 15:52:29
categories: "Tools"
comments: true
featured_image: antlr-logo.png
tags:
- ANTLR
- ANTLR 4
- 语法解析器
---

<!-- no node -->

<!-- more -->

最近因为业务需要，所以开始接触 ANTLR 4 语法解析器，在业务层面需要适配自己业务特征的 DSL 输入框。

调研了几种方向：

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| 纯纯的 `string` 解析 | 开发周期短，轻松 | 迭代成本高，容易推翻重做 |
| Babel 解析 | 基于文本解析成 AST 语法树，通过 AST 来完成业务，迭代轻松 | 需要实现文本分词，浏览器环境使用为动态解析，存在性能损耗 |
| ANTLR 4 解析 | 定义语法能够静态编译出对应代码，迭代轻松 | 需要学习 ANTLR 4 生态环境 |

> 结合业务场景的需要，考虑到未来的迭代可能，所以决定使用 ANTLR 4 解析方案。

# 安装

`package.json` 所需依赖：

| 名称 | 版本 | 说明 |
| --- | --- | --- |
| `antlr4ng-cli` | `^1.0.7` | 将 ANTLR 4 编译为 TypeScript 文件的终端执行文件 |
| `antlr-format-cli` | `^1.2.1` | 美化 `*.g4` 格式 |
| `antlr4ng` | `2.0.11` | ANTLR 4 在 JavaScript 运行时 |
| `antlr4-c3` | `3.3.7` | ANTLR 4 代码补全包 |

> 使用 `antlr4ng-cli` 编译需要电脑安装 JDK ，我这里对应的 JDK 版本是 17.0.12。
> 编译脚本参考[DTStack/dt-sql-parser](https://github.com/DTStack/dt-sql-parser/blob/main/scripts/antlr4.js)。
> 仓库使用 `inquirer` 包进行多 DSL 编译，因为业务场景，我移除了此功能，只对指定的文件进行编译工作。

# 解析器

简单做个示例，实现变量比较的功能。

在对应的目录下创建 `XXLLexer.g4` 词法分析文件和 `XXLParser.g4` 解析文件。

```antlr
// XXLLexer.g4
// ...
lexer grammar XXLLexer;

options {
    caseInsensitive= true;
}

EQUAL                  : '==';
AND                    : 'AND';
OR                     : 'OR';

VALUE                  : (LETTER | DIGIT)+;

SPACE                  : [ \t\r\n]+ -> skip;

fragment DIGIT         : [0-9];
fragment LETTER        : [A-Za-z];
```

```antlr
// XXLParser.g4
// ...
parser grammar XXLParser;

options {
    tokenVocab= XXLLexer;
    caseInsensitive= true;
    superClass=ParserBase;
}

@header {
import { ParserBase } from '../ParserBase';
}

program
    : singleStatement (AND | OR)* singleStatement* EOF
    ;

singleStatement
    : singleStatement (AND | OR) singleStatement
    | equalStatement
    ;

equalStatement
    : VALUE EQUAL VALUE
    ;
```

最后支持的效果是：

- `a == b`
- `C1 == Z AND a == b`

解析器的类也基于业务做了调整，可以参考 dt-sql-parser 项目。
g4 语法学习请参考[Getting Started with ANTLR v4](https://github.com/antlr/antlr4/blob/dev/doc/getting-started.md)。
语法参考[antlr/grammars-v4](https://github.com/antlr/grammars-v4)。

# 代码补全

可以自己基于解析器实现一个代码补全方法，创建的方法：

```typescript
import { CandidatesCollection, CodeCompletionCore } from 'antlr4-c3';

// ...

const core = new CodeCompletionCore(parserIns/* 解析器实例 */);
/**
 * 设置需要匹配的规则
 * 比如：singleStatement
 * 则设置：
 * protected preferredRules = new Set([
 *   XXLParser.RULE_singleStatement,
 * ]);
 */
core.preferredRules = this.preferredRules;

// 获得候选列表
const candidates = core.collectCandidates(caretTokenIndex, c3Context);
```

基于候选列表，可以提供一个体检舒适的 DSL 文本输入框，具备代码补全功能。

到此，算是浅尝了一下 ANTLR 4 ，能感受到的好处就是，当 DSL 产生迭代后，我基本只需要修改 g4 文件，重新编译即可。

非常的舒适～
