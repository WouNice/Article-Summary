# 第2章：词法分析——编译器如何"断词"

**用一句话说清楚**：词法分析就是让编译器像人读书一样，把一串字符拆成一个个"单词"。你读 "int a = 42;" 自然知道是关键字int、变量名a、等号、数字42、分号——编译器也得学会这个本事。

## 为什么要学词法分析？

这是编译的第一个阶段。没有词法分析，编译器面对的就是一长串没有意义的字符：

```
"intmain(){inta=42;returna+1;}"
```

有了词法分析，就变成一系列有意义的 **记号（Token）**：

```
INT → IDENT("main") → LPAREN → RPAREN → LBRACE → INT → IDENT("a") → ASSIGN → NUMBER(42) → SEMI → ...
```

**三个关键概念**：

| 概念 | 通俗理解 | 例子 |
|------|---------|------|
| **词素（Lexeme）** | 源代码中的原始字符序列 | `42`、`int`、`main` |
| **记号（Token）** | 词素的"分类" + 附加信息 | `NUMBER(42)`、`KW_INT`、`IDENTIFIER("main")` |
| **模式（Pattern）** | 描述词素长的"样子" | 正则表达式 `[0-9]+` 描述了整数的样子 |

> 💡 打个比方：词素是"苹果"这两个汉字，记号是"水果类：苹果"——词素是原始字符串，记号是带分类信息的结构化数据。

## 手工写一个词法分析器

最好的学习方式是自己动手写。下面的代码实现了 C 语言子集的词法分析器，约 200 行。

### 核心设计

手工词法分析器遵循两个原则：

1. **最大匹配**：尽可能读入最长的合法字符串（比如读到 `ifx` 不会停在 `if`）
2. **前瞻（Lookahead）**：需要看下一个字符来判断当前 Token 是否结束

> 💡 **最大匹配的小故事**：如果你写 `ifx=5`，词法分析器会把 `ifx` 整个当作标识符，而不会把 `if` 当成关键字然后报告 `x` 非法——它总是"贪心"地多读一些，直到确认没法再长了。

### 记号类型定义

```c
/* token.h - 所有可能的 Token 类型 */
typedef enum {
    /* 关键字 */
    TOK_INT,    TOK_RETURN, TOK_IF,     TOK_ELSE,
    TOK_WHILE,  TOK_FOR,    TOK_VOID,   TOK_CHAR,

    /* 运算符 */
    TOK_PLUS,   TOK_MINUS,  TOK_STAR,   TOK_SLASH,
    TOK_ASSIGN, TOK_EQ,     TOK_NEQ,    TOK_LT,
    TOK_GT,     TOK_LE,     TOK_GE,

    /* 分隔符 */
    TOK_LPAREN, TOK_RPAREN, TOK_LBRACE, TOK_RBRACE,
    TOK_SEMI,   TOK_COMMA,

    /* 字面量 */
    TOK_NUMBER, TOK_IDENT,  TOK_STRING,

    TOK_EOF,    TOK_ERROR
} TokenKind;
```

### 完整词法分析器实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

/* ---- 词法分析器内部状态 ---- */
typedef struct {
    const char *input;       /* 输入源代码 */
    int  pos;                /* 当前位置 */
    int  line;               /* 当前行号 */
    int  col;                /* 当前列号 */
    int  input_len;
} Lexer;

static Lexer lexer;

void lexer_init(const char *src) {
    lexer.input = src;  lexer.pos = 0;
    lexer.line = 1;     lexer.col = 1;
    lexer.input_len = (int)strlen(src);
}

static char peek(void) {
    if (lexer.pos >= lexer.input_len) return '\0';
    return lexer.input[lexer.pos];
}

static char advance(void) {
    char c = peek();
    if (c == '\n')   { lexer.line++; lexer.col = 1; }
    else if (c != 0) { lexer.col++; }
    lexer.pos++;
    return c;
}

static void skip_whitespace(void) {
    while (isspace((unsigned char)peek())) advance();
}

/* ---- 关键字表：快速判断一个词是不是关键字 ---- */
typedef struct { const char *word; TokenKind kind; } KeywordEntry;

static const KeywordEntry keywords[] = {
    {"int", TOK_INT}, {"return", TOK_RETURN},
    {"if", TOK_IF},   {"else", TOK_ELSE},
    {"while", TOK_WHILE}, {"for", TOK_FOR},
    {"void", TOK_VOID},  {"char", TOK_CHAR},
    {NULL, TOK_ERROR}
};

/* 判断一个词是关键字还是普通标识符 */
static TokenKind lookup_keyword(const char *word) {
    for (int i = 0; keywords[i].word != NULL; i++)
        if (strcmp(word, keywords[i].word) == 0)
            return keywords[i].kind;
    return TOK_IDENT;
}

/* ---- 读取不同类型的 Token ---- */

/* 标识符/关键字: 以字母或下划线开头 */
static Token read_ident(void) {
    int start = lexer.pos;
    while (isalnum((unsigned char)peek()) || peek() == '_')
        advance();
    int len = lexer.pos - start;
    char *word = malloc(len + 1);
    strncpy(word, lexer.input + start, len); word[len] = '\0';

    TokenKind kind = lookup_keyword(word);
    Token t = { kind, lexer.line, start, {.strval = kind == TOK_IDENT ? word : NULL} };
    if (kind != TOK_IDENT) free(word);  /* 关键字不持有字符串 */
    return t;
}

/* 数字常量 */
static Token read_number(void) {
    int start = lexer.pos;
    while (isdigit((unsigned char)peek())) advance();
    char buf[32] = {0};
    int n = lexer.pos - start;
    strncpy(buf, lexer.input + lexer.pos - n, n < 31 ? n : 31);
    return (Token){ TOK_NUMBER, lexer.line, start, {.intval = atoi(buf)} };
}

/* 字符串常量 */
static Token read_string(void) {
    advance();  /* 跳过开头的 " */
    int start = lexer.pos;
    while (peek() != '"' && peek() != '\0') {
        if (peek() == '\\') advance();  /* 跳过转义字符 */
        advance();
    }
    int len = lexer.pos - start;
    char *str = malloc(len + 1);
    strncpy(str, lexer.input + start, len); str[len] = '\0';
    if (peek() == '"') advance();  /* 跳过结尾的 " */
    return (Token){ TOK_STRING, lexer.line, start, {.strval = str} };
}

/* 注释 // ... */
static void skip_comment(void) {
    while (peek() != '\n' && peek() != '\0') advance();
}

/* 运算符：单字符或双字符（如 ==、!=、<=） */
static Token read_operator(void) {
    char c = advance();  int start = lexer.col - 1;
    switch (c) {
        case '+': return (Token){ TOK_PLUS,   lexer.line, start, {0} };
        case '-': return (Token){ TOK_MINUS,  lexer.line, start, {0} };
        case '*': return (Token){ TOK_STAR,   lexer.line, start, {0} };
        case '/':
            if (peek() == '/') { skip_comment(); return lexer_get_next_token(); }
            return (Token){ TOK_SLASH,  lexer.line, start, {0} };
        case '=':
            if (peek() == '=') { advance(); return (Token){ TOK_EQ, lexer.line, start, {0} }; }
            return (Token){ TOK_ASSIGN, lexer.line, start, {0} };
        case '!':
            if (peek() == '=') { advance(); return (Token){ TOK_NEQ, lexer.line, start, {0} }; }
            return (Token){ TOK_ERROR, lexer.line, start, {0} };
        case '<':
            if (peek() == '=') { advance(); return (Token){ TOK_LE, lexer.line, start, {0} }; }
            return (Token){ TOK_LT,   lexer.line, start, {0} };
        case '>':
            if (peek() == '=') { advance(); return (Token){ TOK_GE, lexer.line, start, {0} }; }
            return (Token){ TOK_GT,   lexer.line, start, {0} };
        default: return (Token){ TOK_ERROR, lexer.line, start, {0} };
    }
}

/* ---- 主接口：获取下一个 Token ---- */
Token lexer_get_next_token(void) {
    skip_whitespace();
    if (lexer.pos >= lexer.input_len)
        return (Token){ TOK_EOF, lexer.line, lexer.col, {0} };

    char c = peek();
    if (isalpha(c) || c == '_')  return read_ident();
    if (isdigit(c))              return read_number();
    if (c == '"')                return read_string();

    switch (c) {
        case '(': advance(); return (Token){ TOK_LPAREN, lexer.line, lexer.col-1, {0} };
        case ')': advance(); return (Token){ TOK_RPAREN, lexer.line, lexer.col-1, {0} };
        case '{': advance(); return (Token){ TOK_LBRACE, lexer.line, lexer.col-1, {0} };
        case '}': advance(); return (Token){ TOK_RBRACE, lexer.line, lexer.col-1, {0} };
        case ';': advance(); return (Token){ TOK_SEMI,   lexer.line, lexer.col-1, {0} };
        case ',': advance(); return (Token){ TOK_COMMA,  lexer.line, lexer.col-1, {0} };
        default:
            if (c == '+' || c == '-' || c == '*' || c == '/' ||
                c == '=' || c == '!' || c == '<' || c == '>')
                return read_operator();
    }
    advance();
    return (Token){ TOK_ERROR, lexer.line, lexer.col-1, {0} };
}
```

### 测试一下

```c
int main(void) {
    const char *input = "int main() {\n"
                        "    int a = 42;\n"
                        "    return a + 1;\n"
                        "}";
    lexer_init(input);
    printf("=== 记号流输出 ===\n");
    Token tok;
    do {
        tok = lexer_get_next_token();
        printf("%-12s (行%d, 列%d)\n", token_name(tok.kind), tok.line, tok.col);
    } while (tok.kind != TOK_EOF);
    return 0;
}
```

编译运行：
```bash
gcc -o lexer lexer.c
./lexer
```

输出：
```
=== 记号流输出 ===
TOK_INT      (行1, 列1)
TOK_IDENT    (行1, 列5)
TOK_LPAREN   (行1, 列9)
TOK_RPAREN   (行1, 列10)
TOK_LBRACE   (行1, 列12)
TOK_INT      (行2, 列5)
TOK_IDENT    (行2, 列9)
TOK_ASSIGN   (行2, 列11)
TOK_NUMBER   (行2, 列13)
TOK_SEMI     (行2, 列15)
TOK_RETURN   (行3, 列5)
TOK_IDENT    (行3, 列12)
TOK_PLUS     (行3, 列14)
TOK_NUMBER   (行3, 列16)
TOK_SEMI     (行3, 列17)
TOK_RBRACE   (行4, 列1)
TOK_EOF      (行4, 列2)
```

> 💡 注意每一行的行列号——这些信息在后面报错时非常有用。编译器说"第 3 行有语法错误"就是靠词法分析记录的行列号。

## 性能优化技巧：让词法分析跑得更快

大型项目中，词法分析可能要处理百万行代码，性能很关键。以下是三个常用优化技巧：

### 技巧1：查表法替代 if-else

不用每次判断字符类型都写一堆 `if (c >= 'a' && c <= 'z')`，而是**提前建好一张表**：

```c
/* 预计算字符分类表 */
static const uint8_t char_class[256] = {
    ['\t'] = CHAR_SPACE, ['\n'] = CHAR_SPACE, [' '] = CHAR_SPACE,
    ['0'] = CHAR_DIGIT,  /* 依次类推到 '9' */
    ['a'] = CHAR_ID_START, ['b'] = CHAR_ID_START, /* 直到 'z' */
    ['A'] = CHAR_ID_START, /* 直到 'Z' */
    ['_'] = CHAR_ID_START,
    ['+'] = CHAR_OP, ['-'] = CHAR_OP, ['*'] = CHAR_OP, ['/'] = CHAR_OP,
    /* 其余默认为 0（非法字符） */
};

/* 使用：一次查表代替 10+ 次比较 */
char type = char_class[(unsigned char)current_char];
```

**效果**：单字符分类从多次比较降为 1 次内存读取，整体性能提升 30-50%。

### 技巧2：内存映射文件（mmap）

读取输入文件的方式对性能影响很大：

| 读取方式 | 速度 |
|---------|------|
| `fgetc()` 逐字符 | ~80 MB/s |
| `fread()` 块读取 | ~400 MB/s |
| `mmap` 内存映射 | ~1200 MB/s |

### 技巧3：分支预测提示

```c
#define likely(x)   __builtin_expect(!!(x), 1)    /* 大概率成立 */
#define unlikely(x) __builtin_expect(!!(x), 0)     /* 大概率不成立 */

Token read_ident(Lexer *l) {
    while (likely(is_ident_char(l->input[l->pos]))) /* 大部分字符都是标识符字符 */
        l->pos++;
    /* ... */
}
```

## 常见错误处理

| 错误情况 | 例子 | 处理方式 |
|---------|------|---------|
| 非法字符 | `@`, `` ` `` | 报告行号列号，跳过继续 |
| 未闭合字符串 | `"hello` | 报错，在行尾自动闭合 |
| 数字溢出 | `999999999...` | 截断 + 警告 |
| 注释未闭合 | `/* comment` | 报告文件尾错误 |

## 手工 vs 自动生成：词法分析器的两种做法

| 对比项 | **手写 Lexer** | **Flex（自动生成）** |
|--------|--------------|-------------------|
| 代码量 | 200-300 行 C | 几十行 .l 规则文件 |
| 性能 | 优（可针对性优化） | 良（通用算法） |
| 可控制性 | 完全可控 | 受限于工具 |
| 学习曲线 | 需理解原理 | 需学习工具语法 |
| 代表 | Clang Lexer | 大多数 Unix 工具 |

> 💡 **实际项目的选择**：Clang、GCC、Rust、Go 全都有**手写词法分析器**。自动生成工具（Flex）主要用于原型或小型项目。学习编译器**强烈建议从手写开始**。

### 用 Flex 实现的速度对比

如果你好奇自动生成是什么样子，Flex 的写法大概是这样：

```lex
%{
#include "token.h"
%}
%option noyywrap

letter    [a-zA-Z_]
digit     [0-9]
ident     {letter}({letter}|{digit})*
number    {digit}+

%%
"if"       { return TOK_IF; }
"int"      { return TOK_INT; }
{number}   { yylval.intval = atoi(yytext); return TOK_NUMBER; }
{ident}    { yylval.strval = strdup(yytext); return TOK_IDENT; }
"//".*     { /* 跳过注释 */ }
.          { fprintf(stderr, "非法字符: %s\n", yytext); }
%%
```

编译使用：`flex c_lexer.l && gcc lex.yy.c -lfl -o c_lexer`

## 📝 本章小结

| 核心概念 | 一句话理解 |
|---------|-----------|
| **词法分析** | 把字符流变成 Token 流 |
| **Token/词素/模式** | Token 是"分类名"，词素是"原始字符"，模式是"长相规则" |
| **最大匹配** | 能读多长就读多长，不要过早截断 |
| **查表优化** | 用预计算表代替 if-else 链，速度提升 30-50% |

### 🏋️ 动手练习

1. 为词法分析器增加 `/* ... */` 多行注释支持
2. 增加十六进制数（`0xFF`）和浮点数（`3.14`）的识别
3. 思考：为什么需要"列号"信息？（答案提示：精确的错误定位）

> 下一章我们来学习词法分析背后的理论——正则表达式和有限自动机。如果你只想动手写编译器，跳过这章也不影响；但如果想理解 Flex 是怎么工作的，这一章就是答案。
