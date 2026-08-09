# 第8章：目标代码生成——IR 变机器码

**用一句话说清楚**：经过前面 6 个阶段，编译器有了优化后的 IR，现在是最后一步——把 IR 翻译成 CPU 能执行的机器指令。就像翻译官把演讲稿从"世界语"翻成"英语"。

## 从 TAC 到 x64 汇编

### 核心任务

| TAC 指令 | → | x64 汇编 |
|----------|---|----------|
| `t1 = a + b` | → | `movl a, %eax; addl b, %eax; movl %eax, t1` |
| `if t < 10 goto L` | → | `cmpl $10,%eax; jge L` |
| `goto L` | → | `jmp L` |
| `call f, n` | → | `call f` |
| `return x` | → | `movl x, %eax; ret` |

> 💡 TAC 是"抽象机器"的指令，真实 CPU 的指令更具体也更复杂——x64 有更多寻址模式、条件码寄存器、调用约定等。

### 从 TAC 到 x64 的翻译模式

```c
// 核心：简单的一一映射
void gen_code(TAC *tac) {
    switch (tac->kind) {
        case TAC_BINOP: {
            // t1 = a + b
            gen_load(tac->arg1, EAX);     // 把 a 装入 eax
            gen_mem_to_reg(tac->arg2, ECX); // 把 b 装入 ecx
            fprintf(out, "    addl %%ecx, %%eax\n");  // eax = eax + ecx
            gen_store(EAX, tac->result);  // 结果存到 t1
            break;
        }
    }
}
```

但**实际情况要复杂得多**——好的代码生成器会做指令选择（选择最合适的 x64 指令）、寻址模式利用、指令合并等。

## 指令选择：TAC 到汇编不是一一对应

一个好的代码生成器会把多个 TAC 合并成一条**更高效**的汇编指令：

```text
TAC：                   简单翻译：               优化翻译：
    t1 = a + b              movl a, %eax             movl a, %eax
    t2 = t1 + c             addl b, %eax             addl b, %eax
    d = t2                  movl %eax, t1            addl c, %eax
                            movl t1, %eax            movl %eax, d  ← 少了2条！
                            addl c, %eax
                            movl %eax, t2
                            movl t2, %eax
                            movl %eax, d
```

> 💡 **10 条指令 vs 4 条指令**——好的指令选择能**节省 60% 以上**的指令数。

## 函数调用约定

编译器需要在**调用者和被调用者之间达成协议**——参数放哪里？返回值放哪里？

以 x64 Linux 的 System V ABI 为例：

| 参数位置 | 寄存器/栈 |
|---------|----------|
| 第1-6个参数 | rdi, rsi, rdx, rcx, r8, r9 |
| 第7+个参数 | 压栈（从右向左） |
| 返回值 | rax |
| 调用者保存 | rcx, rdx, rsi, rdi, r8-r11 |
| 被调用者保存 | rbx, rbp, r12-r15 |

> 💡 **为什么编译器必须遵守这个约定？** 因为你的 C 程序里调用了 `printf`——printf 是其他编译器（或 libc）编译的，它期望参数在 rdi、rsi 等寄存器里。如果你的编译器把参数放在别的寄存器，程序就会崩溃。

## 寄存器分配：图着色算法

这是后端最重要的优化之一。把 N 个变量分配到 M 个寄存器（M 通常很小，x64 通用寄存器约 16 个）。

**核心思路**：
1. 建**冲突图**：同时活跃的变量之间有"冲突边"
2. 图着色：给每个变量"染"上寄存器颜色，冲突的变量不能同色

```text
冲突图示例：
    变量活跃区间：
    a: ──────────
    b:    ──────────
    c:       ──────
    d:  ──

    冲突边: a-b, a-c, b-c, b-d
    染色结果: a→%eax, b→%ebx, c→%ecx, d→%eax (a 和 d 不冲突，共用寄存器)
```

## 指令调度：利用 CPU 流水线

现代 CPU 可以同时执行多条指令，只要它们之间没有**数据依赖**：

```text
源代码：                 指令调度后（更快的版本）：
    a = x + y              t1 = x * z       ← 两条指令互不依赖
    b = x * z              a = x + y        ← 可以并行执行！
    c = a + b              c = a + b        ← 等上面两行结果
```

> 💡 CPU 硬件会自动做一些调度（乱序执行），但编译器做的调度比硬件更好——编译器可以看到更大范围的指令。

## 汇编输出示例

TinyC 代码 → 生成的 x64 汇编：

```c
int add(int a, int b) {
    int c = a + b;
    return c;
}
```

```asm
add:
    pushq   %rbp                # 保存栈帧基址
    movq    %rsp, %rbp          # 建立新栈帧
    movl    %edi, -4(%rbp)      # 第一个参数 a → 栈上局部变量
    movl    %esi, -8(%rbp)      # 第二个参数 b → 栈上局部变量
    movl    -4(%rbp), %eax      # 取出 a
    addl    -8(%rbp), %eax      # 加 b
    movl    %eax, -12(%rbp)     # 存 c
    movl    -12(%rbp), %eax     # 恢复返回值到 eax
    popq    %rbp                # 恢复栈帧
    ret                         # 返回
```

**经过优化后**（寄存器分配生效）：

```asm
add:
    movl    %edi, %eax          # a → eax（不用内存）
    addl    %esi, %eax          # a + b → eax
    ret                         # 直接返回！
```

> 💡 从 10 条指令降到 **2 条**——寄存器分配让代码直接从寄存器操作，完全绕过内存。

## 📝 本章小结

| 概念 | 一句话理解 |
|------|-----------|
| **目标代码生成** | 把 IR 翻译成具体 CPU 的机器码 |
| **指令选择** | 选最合适的汇编指令，一条汇编可能抵多条 TAC |
| **寄存器分配** | 把热门变量分配给最快的内存（寄存器） |
| **调用约定** | 函数间关于参数/返回值存放位置的"协议" |
| **指令调度** | 重排不相关的指令来利用 CPU 的并行能力 |

### 🏋️ 动手练习

1. 把第 1 章微型编译器的代码生成器升级为生成更多优化指令
2. 用 `echo 'int f(int x){return x+42;}' | gcc -O2 -S -xc -o - -` 查看优化后汇编
3. 尝试理解 CMOV（条件移动）指令——它能避免分支预测失败的开销

> 下一章是本书的重头戏——手写一个完整的 TinyC 编译器，把前面 8 章的知识串起来！
