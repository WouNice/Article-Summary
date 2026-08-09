# 第9章：完整编译器实战——手写 TinyC

**用一句话说清楚**：前面 8 章你学了编译器的每一个零部件，这一章把它们全部组装起来——一个约 1000 行的 C 语言子集编译器。读完它，你会真正理解"编译器就是这么回事"。

## TinyC 的约束

**TinyC** 是一个完整的 C 语言子集编译器，支持：

| 特性 | 状态 |
|------|------|
| ✅ 函数定义（含返回值类型） | 支持 `int f(int x) { ... }` |
| ✅ 变量声明（支持多个） | 支持 `int a, b, c;` |
| ✅ 算术表达式（+-*/） | 支持运算符优先级 |
| ✅ if-else 语句（支持嵌套） | 支持 `if/else if/else` |
| ✅ while 循环 | 支持 `while (expr) { ... }` |
| ✅ 函数调用 | 支持参数传递、返回值 |
| ✅ 局部变量 | 栈上分配 |
| ✅ 递归 | 支持 |
| ❌ 指针 | 不支持 |
| ❌ 数组 | 不支持 |
| ❌ 结构体 | 不支持 |
| ❌ switch | 不支持 |

## 架构总览

```
                        TinyC 架构
          ┌─────────────────────────────────┐
          │     main.c (驱动程序)             │
          └─────────────────────────────────┘
                      ↓
          ┌─────────────────────────────────┐
          │    lexer.c (词法分析器)           │
          │    源文件 → Token 流              │
          └─────────────────────────────────┘
                      ↓
          ┌─────────────────────────────────┐
          │    parser.c (语法分析器 + AST)    │
          │    Token 流 → 语法树              │
          └─────────────────────────────────┘
                      ↓
          ┌─────────────────────────────────┐
          │    semantic.c (语义分析)          │
          │    类型检查 + 作用域管理           │
          └─────────────────────────────────┘
                      ↓
          ┌─────────────────────────────────┐
          │    codegen.c (代码生成)           │
          │    AST → x64 汇编                │
          └─────────────────────────────────┘
                      ↓
                    out.s → as + ld → 可执行文件
```

## 词法分析器（Lexer）

与第 2 章类似，TinyC 的 lexer 也是查表驱动的手写词法分析器。核心结构：

```c
typedef enum {
    TK_IDENT, TK_NUMBER,
    TK_INT, TK_VOID, TK_IF, TK_ELSE, TK_WHILE, TK_RETURN,
    TK_PLUS, TK_MINUS, TK_STAR, TK_SLASH,
    TK_ASSIGN, TK_EQ, TK_NEQ, TK_LT, TK_GT, TK_LE, TK_GE,
    TK_LPAREN, TK_RPAREN, TK_LBRACE, TK_RBRACE, TK_SEMI, TK_COMMA,
    TK_EOF
} TokenKind;

/* 关键字识别：哈希表存储，O(1) 查找 */
TokenKind lookup_keyword(const char *name) {
    int index = hash(name, 17);      // 取模哈希
    return kw_table[index];           // O(1) 查表
}
```

**特点是哈希表关键字查找**——不遍历数组，而是用预计算的哈希表，O(1) 时间判断是关键字还是标识符。

## 语法分析器（Parser）

TinyC 使用递归下降分析，并为 C 语言的典型语句每个写一个函数：

```c
typedef struct ASTNode {
    NodeKind kind;                  // 节点类型
    Type *type;                     // 表达式类型
    union {
        struct { int value; } num;           // 数字
        struct { char *name; } ident;        // 标识符
        struct { ASTNode *cond, *then, *el; } if_stmt;  // if
        struct { ASTNode *cond, *body; } while_stmt;    // while
        struct { ASTNode *left, *right; } binary;       // 二元运算
        struct { ASTNode *operand; } unary;             // 一元运算
        struct { char *name; ASTNode **args; int nargs; } call;  // 函数调用
    };
} ASTNode;
```

核心文法（用 BNF 描述）：

```text
program    = { function_definition }
function   = type IDENT '(' params ')' '{' stmts '}'
stmt       = '{' stmts '}' | if_stmt | while_stmt | return_stmt | expr_stmt
stmts      = { var_decl | stmt }
expr       = assign_expr
assign_expr = equality_expr { '=' assign_expr }
equality   = relational_expr { ('=='|'!=') relational_expr }
relational = add_expr { ('<'|'>'|'<='|'>=') add_expr }
add_expr   = mul_expr { ('+'|'-') mul_expr }
mul_expr   = unary_expr { ('*'|'/') unary_expr }
unary_expr = ('-'|'!') unary_expr | primary
primary    = NUMBER | IDENT | IDENT '(' args ')'
           | '(' expr ')' | STRING
```

### 类型系统

```c
typedef struct Type {
    TypeKind kind;      // INT, VOID, FUNCTION
    struct {
        Type *ret;      // 函数返回类型
        Type **params;  // 函数参数类型列表
        int nparams;    // 参数个数
    } func;
    int size;           // sizeof 大小
    int align;          // 对齐要求
} Type;
```

## 代码生成器（Codegen）

TinyC 的主要创新——**对 while 循环使用逆向遍历**，保证先输出循环体再输出条件，这样循环的开头地址已经确定：

```c
// 逆向遍历 while 语句
void gen_while(ASTNode *node) {
    // 1. 先生成循环体——确定 L_body 地址
    gen_stmt(node->while_stmt.body);
    // 此时 L_body = current_pos

    // 2. 再计算条件——这里可以正确填充跳转目标
    gen_cond(node->while_stmt.cond);
    // ...

    // 3. 生成条件跳转回 L_body
    // jmp L_body
}
```

函数调用的代码生成模式：

```c
// 参数约定（System V x64 ABI）：
//   第 1-6 个参数 → %rdi, %rsi, %rdx, %rcx, %r8, %r9
//   额外参数 → 栈上
//   返回值 → %rax
void gen_func_call(ASTNode *node) {
    for (int i = 0; i < node->call.nargs; i++) {
        gen_expr(node->call.args[i]);     // 生成参数求值
        // 将结果移到正确的寄存器
        fprintf(out, "    movl %%eax, %%e%s\n", ARG_REG[i]);
    }
    fprintf(out, "    call %s\n", node->call.name);
    // 返回值已在 %eax 中
}
```

## 完整例子：斐波那契数列

输入 TinyC 代码：

```c
int fib(int n) {
    if (n <= 1) {
        return 1;
    } else {
        return fib(n-1) + fib(n-2);
    }
}

int main(void) {
    int result;
    result = fib(10);
    return result;
}
```

生成的 x64 汇编（精简后）：

```asm
fib:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $8, %rsp              # 分配局部变量空间

    cmpl    $1, %edi              # if (n <= 1) ...
    jg      .L_else
    movl    $1, %eax
    jmp     .L_end

.L_else:
    # fib(n-1)
    movl    %edi, -4(%rbp)        # 保存 n
    subl    $1, %edi              # n-1
    call    fib
    movl    %eax, -8(%rbp)        # 存左子树结果

    # fib(n-2)
    movl    -4(%rbp), %edi        # 恢复 n
    subl    $2, %edi              # n-2
    call    fib

    addl    -8(%rbp), %eax        # 左 + 右

.L_end:
    movq    %rbp, %rsp
    popq    %rbp
    ret
```

编译运行：

```bash
# 用 tinycc 编译 fib.tc
./tinycc fib.tc out.s

# 用系统汇编器汇编
gcc -o fib out.s -no-pie

# 运行
./fib
echo $?    # 输出: 89（Fibonacci(10) = 89）
```

## 编译 TinyC 编译器自身的尝试

TinyC 的一个重要特性是它能编译自身——我们称之为**自举（bootstrapping）**。

```bash
# 第一步：用系统 GCC 编译 TinyC
gcc -o tinycc_s1 tinyc.c

# 第二步：用 tinycc_s1 编译 tinyc.c
./tinycc_s1 tinyc.c out1.s
gcc -o tinycc_s2 out1.s -no-pie

# 第三步：用第二阶段编译器再次编译
./tinycc_s2 tinyc.c out2.s
gcc -o tinycc_s3 out2.s -no-pie

# 验证：比较 tinycc_s3 和 tinycc_s2 是否一致（行为等价）
```

> 💡 **自举是编译器开发中一个迷人的里程碑**——意味着你的编译器不再依赖其他编译器，可以独立编译自己。

## 扩展练习：迷你优化器

在代码生成后，可以添加一个简单的**窥孔优化器（Peephole Optimizer）**——扫描生成的汇编，替换局部低效模式：

```c
// 窥孔优化规则
void peephole_optimize(char *asm_lines[], int nlines) {
    // 规则1: mov %r, %r  → 删除（自己移给自己没有意义）
    if (match("movl %%r%d, %%r%d", r1, r2) && r1 == r2) { delete(); }

    // 规则2: push; pop  → 删除（压栈后立刻弹出等于白做）
    if (match("push %%r%d", r) && match_next("pop %%r%d", r)) { delete(2); }

    // 规则3: mov $c, %r; add $c, %r → add $2*c, %r（合并常量操作）
    if (match("movl $%d, %%r%d", v1, r) &&
        match_next("addl $%d, %%r%d", v2, r)) {
        replace_with("addl $%d, %%r%d", v1+v2, r);
    }
}
```

这些是**最常见的三种窥孔优化模式**——它们在真实编译器中极为有效。

## 📝 本章小结

| 组件 | TinyC 实现策略 |
|------|---------------|
| **Lexer** | 手写查表法，关键字哈希查找 |
| **Parser** | 递归下降，每个语法成分一个函数 |
| **Semantic** | 类型检查 + 符号表作用域链 |
| **Codegen** | AST → x64 汇编，逆向遍历处理 while |
| **自举** | 可用自身编译自身 |

### 🏋️ 动手练习

1. 为 TinyC 添加 `for` 循环支持（在 parser 中添加 for 语法分支 + codegen 添加 for 翻译）
2. 添加一元 `&` 取地址运算符和 `*` 解引用运算符（第 9.1 节 ⚠️ 的进阶挑战）
3. 实现 `printf` 调用的参数类型自动匹配

> 下一章收集了编译原理的各个高级专题——深入 Flex、Yacc、LLVM 的细节，以及一些"锦上添花"的知识点。
