# Clang 优化能力有多厉害：一个 bug 耗了我两天时间

> 一段 `[[gnu::returns_twice]]` 的标注，三种编译器的反汇编，把一个断断续续调了两天的
> fiber bug 钉死成"这是编译器之间的差异，不是平台之间的差异"。

最近在做 [flare][flare] 的 macOS 移植，遇到一个相当顽固的 bug：fiber 的测试在 macOS
（Apple Silicon）上一跑就 abort，错误信息长这样：

```text
F fiber_entity.cc:204] Check failed: caller == GetCurrentFiberEntity()
                       SetCurrentFiberEntity did not stick.
```

`caller` 是函数局部变量，`GetCurrentFiberEntity()` 通过一个 `thread_local` 读出"当前
fiber"。`SetCurrentFiberEntity(caller)` 刚把 `caller` 写进 TLS，下一行立刻把它读出来
比较——居然不相等。意味着：**fiber 切换之后，写进去的 `thread_local` 槽和读出来的
`thread_local` 槽不是同一个槽。**

[flare]: https://github.com/Tencent/flare

调查路径相当曲折——前后断断续续花了两天才定位下来（中间穿插着别的活儿，不然这事不
至于拖这么久）。先后试过统一 `[[gnu::returns_twice]]` 声明、`volatile thread_local`、
`asm volatile("" ::: "memory")`——三个看起来都该是"正解"的修复，**没有一个能让 Clang
让出 `x19`**。最后定位到的结论比"macOS 有 bug"精确得多：**Clang 在 AArch64 上把 TLS 槽
的地址跨过 `returns_twice` 函数做了 CSE（common subexpression elimination，公共子表达
式消除——把"算出同样结果"的多次计算合并成一次复用），而 GCC 不会。** Linux 历史上没出
问题，是 GCC 在救我们，不是 Linux 本身比 macOS 安全。

这篇笔记把这个 bug 的来龙去脉记一下，包括三条没走通的弯路、关键的反汇编对比、以及"为什么
`returns_twice`、`volatile`、`asm("" ::: "memory")` 都不管用"的精确分析。文末附了用来
跑出本文里那些反汇编的最小源码，照着用三种工具链分别编一下就能复现。

---

## 一、背景：fiber 怎么把同一份代码运行在不同的 OS 线程上

flare 的 fiber 是一种协作式协程：一个 pthread worker 上跑着很多 fiber，调度器在合适的
时候用 `jump_context` 把当前 fiber 的寄存器/栈状态保存下来，把另一个 fiber 的状态恢复上
去——和 `setjmp/longjmp` 是同一类原语。

为了让 fiber 内部的代码能拿到"我现在是哪个 fiber"，flare 把当前 fiber 指针放在
`thread_local`：

```cpp
extern thread_local FiberEntity* current_fiber;

FiberEntity* GetCurrentFiberEntity() { return current_fiber; }
void         SetCurrentFiberEntity(FiberEntity* p) { current_fiber = p; }
```

这里有一个微妙但关键的语义：

> **`jump_context` 可能把当前 C++ 函数帧恢复在另一个 OS 线程上。**

flare 的调度器是 work-stealing 的，一个 fiber 在 worker A 上挂起、在 worker B 上被唤
醒，是再正常不过的事情。这意味着：跨过 `jump_context` 那一刻，**`thread_local` 槽对
应的虚拟地址变了**——之前那个槽属于 A 这个 OS 线程的 TLS 区，恢复之后应当指向 B 的
TLS 区。同一个进程的地址空间里，每个 OS 线程各有一份独立的 thread-local 存储，编译
器为某个 `thread_local` 变量算出来的具体那个槽，是当前线程的 TLS base 加上一个固定
的偏移。"当前线程"变了，槽地址自然就变了。

C/C++ 标准里没有任何属性可以表达"这个调用可能让我换一个 OS 线程跑"。最接近的是 GCC/
Clang 都支持的 `[[gnu::returns_twice]]`——这是 `setjmp` 和 `vfork` 用的那个属性。它告
诉编译器：

> 这个函数和普通函数不一样，可能从同一个调用点**返回多次**。第一次是"正常返回"，
> 后续返回可能是别的代码（比如 `longjmp`）"跳"回来的——所以调用方在跨过它之后，不
> 能假设本地变量和寄存器的值还是调用前那个，得保守对待。

听起来很贴切——`jump_context` 就是从 `_setjmp` 那个家族派生出来的协程切换原语，第二
次"返回"对应的就是协程被再次唤醒的时刻。所以 flare 给它打上了这个属性：

```cpp
[[gnu::returns_twice]] void jump_context(void** self, void* to, void* context);
```

但 `returns_twice` 的设计是为 `setjmp` 服务的。`setjmp` / `longjmp` 整个生命周期都在
**同一个 OS 线程**里——所以"线程相关的东西在调用前后不变"是 spec 默认成立的前提。
fiber 偏偏违反了这个前提，而 spec 又没给我们一个"线程也可能变"的更强属性。我们以为
`returns_twice` 够用，事后看，并不够。

---

## 二、症状：`SetCurrentFiberEntity did not stick`

在 macOS Apple Silicon 上，`this_fiber_test` 一跑就 SIGABRT，日志大致长这样：

```text
F fiber_entity.cc:204] Check failed: caller == GetCurrentFiberEntity()
                       SetCurrentFiberEntity did not stick.
```

肇事代码就是 [flare/fiber/detail/fiber_entity.cc][src] 里的 `FiberEntity::Resume()`：

```cpp
void FiberEntity::Resume() noexcept {
  auto caller = GetCurrentFiberEntity();                          // ① 读 TLS
  // ...
  auto caller_before = caller;
  jump_context(&caller->state_save_area, state_save_area, this);  // ② 切上下文
                                                                  //    可能换 OS 线程
  FLARE_CHECK_EQ(caller_before, caller, "...");                   // ✓ 局部变量没变

  SetCurrentFiberEntity(caller);                                  // ③ 写 TLS
  FLARE_CHECK_EQ(caller, GetCurrentFiberEntity(),                 // ④ 再读 TLS
                 "SetCurrentFiberEntity did not stick.");         //    ←── 这里 abort
}
```

`caller` 是个普通的局部变量，跨过 `jump_context` 值没变（紧跟 ② 之后的那个
`FLARE_CHECK_EQ(caller_before, caller)` 是过的）。但 ③ 写完 TLS、紧接着 ④ 立刻读出
来，居然不相等。

这只能是：编译器在 ③ 和 ④ 之间**复用了某个"早已拿到的 TLS 槽地址"**——也就是说，
③ 写进去的那个槽，跟 ④ 读出来的那个槽，**不是同一个槽**。

[src]: https://github.com/Tencent/flare/blob/master/flare/fiber/detail/fiber_entity.cc

---

## 三、第一次猜想：`noinline` 是不是其实不必要？

原始代码里这三个 getter/setter 上是带 `[[gnu::noinline]]` 的：

```cpp
[[gnu::noinline]] FiberEntity* GetMasterFiberEntity() noexcept { return master_fiber; }
[[gnu::noinline]] FiberEntity* GetCurrentFiberEntity() noexcept { return current_fiber; }
[[gnu::noinline]] void SetCurrentFiberEntity(FiberEntity* p) { current_fiber = p; }
```

而 `jump_context` 的声明在不同的 `.cc` 里各写了一遍——也就是说，**有些 TU 看到的
`jump_context` 是带 `[[gnu::returns_twice]]` 的，有些没有**。我把声明统一收进了
`context.h`：

```cpp
// flare/fiber/detail/context.h
extern "C" {
  [[gnu::returns_twice]] void jump_context(void** self, void* to, void* context);
  void* make_context(void* sp, std::size_t size, void (*start_proc)(void*));
}
```

直觉上，既然现在每个 TU 都看到了 `returns_twice`，编译器应该乖乖处理 TLS 读写、不再 CSE
跨过 `jump_context`。那 `noinline` 是不是可以拿掉了？

实测：**不行**。把 `[[gnu::noinline]]` 去掉，`this_fiber_test` 立刻 abort 在
`Check failed: self == GetCurrentFiberEntity()` 上。

第一次猜想错。

---

## 四、第二次猜想：`volatile thread_local` 行不行？

如果 Apple Clang 把 TLS 槽地址在寄存器里缓存了，那把变量声明成 `volatile`，让编译器**每
次访问都重读**，应该够吧？

```cpp
extern thread_local FiberEntity* volatile master_fiber;
extern thread_local FiberEntity* volatile current_fiber;
```

实测：**还不行**。同样的 `Check failed` 在同一个地方再 abort 一次。

到这一步我才开始怀疑自己对"Clang 缓存了什么"的理解不够精确。`volatile` 强制了**值的
load/store**，但如果编译器缓存的是"TLS 槽的地址"，那 volatile 只是逼着它"用这个（陈旧的）
地址再读一遍"——读到的还是上一个 OS 线程的槽。

`volatile` 没能解决问题这件事，反而是诊断上的关键信号。

---

## 五、第三次猜想：编译器内存屏障 `asm volatile("" ::: "memory")` 行不行？

`volatile thread_local` 不行，那退一步：在 `jump_context` 之后加一个**编译器级别的内存
屏障**——告诉编译器"屏障之后，内存里任何字节都可能被改过了，所有缓存都得作废"——这
个总该够了吧？

```cpp
jump_context(&caller->state_save_area, state_save_area, this);
asm volatile("" ::: "memory");   // ←── 候选 fix
SetCurrentFiberEntity(caller);
FLARE_CHECK_EQ(caller, GetCurrentFiberEntity(), "...");
```

把最小复现里加上同样的 barrier，重新跑反汇编：

```asm
; Apple Clang，macOS arm64，加了 asm volatile("" ::: "memory") 之后
    mov  x19, x0                  ; 槽地址依然缓存到 x19
    ldr  x20, [x0]
    bl   _jump_context
    ; InlineAsm Start              ; ←── memory barrier 在这（空指令）
    ; InlineAsm End
①   str  x20, [x19]               ; ←── 关键：barrier 之后还是用缓存的 x19！
    ret
```

```asm
; Clang，aarch64-linux-gnu，同样
    add  x19, x8, :tprel_lo12_nc:current_fiber   ; 槽地址缓存到 x19
    ldr  x20, [x19]
    bl   jump_context
    //APP                                         ; ←── memory barrier
    //NO_APP
①   str  x20, [x19]                              ; ←── 关键：barrier 之后还是用缓存的 x19
    ret
```

**Clang 完全无视了 barrier**。barrier 前后 `x19` 没有任何变化，地址依然在用同一个寄存器
里的旧值。

为什么？`asm volatile("" ::: "memory")` 的语义只覆盖**内存内容**——它说的是"屏障之后
任何内存位置可能被改过了，请重新从内存里 load"。但是 `x19` 里**装的不是从内存 load 来
的值**，而是一个**计算结果**：

```text
x19 = TPIDR_EL0 + 链接器解析的偏移        ; Linux Clang
x19 = TLV thunk 的返回值                  ; macOS Apple Clang
```

从编译器看，`x19` 是个"系统寄存器 + 常量"派生出来的纯计算，**不是内存 load**。"memory"
clobber 没有任何理由让它失效——和 `volatile` 失败的根因**完全一样**，都是因为这两个
机制覆盖的是"值"，覆盖不到"地址"。

要让 inline asm 真的解决这个问题，得显式 clobber 槽地址所在的那个寄存器：

```cpp
asm volatile("" ::: "memory", "x19");   // 不可移植，而且基于猜寄存器分配
```

但这就开始猜寄存器分配了——AArch64 今天放 x19，明天的编译器版本或者别的 ABI 上可能放
x20、x21、x28；x86_64 又是另一套。要做对得把所有 GP 寄存器都列出来 clobber，相当于自
己手写一个伪 setjmp 返回——这正是 GCC 内部对 `returns_twice` 已经在做的事，自己再写
一遍又笨又脆。

第三次猜想也错。这时候终于必须把反汇编拿出来看了。

---

## 六、回到真实代码：定罪 `x19`

前面三节的反汇编都跑在最小复现上，已经看到了 Clang 把槽地址往 `x19` 里塞这个动作。
但要把"这就是真凶"钉死，得回到真实代码——把 `flare/fiber/detail/fiber_entity.cc` 里
`Resume()` 那个函数本身的反汇编打出来，看 `jump_context` 前后的 `x19` 到底活到了哪
里、又被谁继续在用。

先看出问题的"inlined 版本"——`Resume()` 函数里 `jump_context` 前后那一段（macOS
Apple Clang，`[[gnu::noinline]]` 已去掉）：

```asm
    adrp x0, _current_fiber@TLVPPAGE
    ldr  x0, [x0, _current_fiber@TLVPPAGEOFF]
    ldr  x8, [x0]
①   blr  x8                       ; ←── 关键：调一次 TLV thunk，拿到槽地址（在 x0）
②   mov  x19, x0                  ; ←── 关键：把槽地址 *缓存* 到 callee-saved x19
    ldr  x20, [x0]                ;       caller = *槽

③   bl   _jump_context            ; ←── 关键：切上下文（可能落到别的 OS 线程）
                                  ;     之后 x19 *仍然*被当作"槽地址"使用

④   str  x20, [x19]               ; ←── 关键：SetCurrentFiberEntity(caller)
                                  ;     往 *旧线程* 的槽里写
⑤   ldr  x8,  [x19]               ; ←── 关键：GetCurrentFiberEntity()
                                  ;     从 *旧线程* 的槽里读
    cmp  x20, x8
    b.ne abort_path               ;       → 抛 Check failed
```

整个反汇编里**只有一次** TLV thunk 调用（① 的 `blr x8`），它返回的槽地址被存到
callee-saved 寄存器 `x19` 里（②）。`x19` 按 AArch64 ABI 是被调用方保存的，
`jump_context` 调用之后 caller 看到的 `x19` 跟调用前一模一样——但实际上**线程已经换
了，那个地址指的还是上一个 OS 线程的槽**。④ 写、⑤ 读，都是用这个"上一个线程的"地
址在操作，自然就崩了。

再看 `[[gnu::noinline]]` 加回去之后的版本——每次访问 `current_fiber` 都是一次实打实的
函数调用：

```asm
    bl  _GetCurrentFiberEntity    ; caller = GetCurrentFiberEntity()
    bl  _jump_context             ; 切上下文
    bl  _SetCurrentFiberEntity    ; SetCurrentFiberEntity(caller)
    bl  _GetCurrentFiberEntity    ; ←── 关键：CHECK_EQ 时再次进 callee
                                  ;     callee 内部重新解析 TLV thunk
                                  ;     拿到 *当前线程* 的槽地址
    cmp ...
```

ABI 规定调用方不能假设外部函数的返回值在两次调用之间相等。所以每个
`bl _GetCurrentFiberEntity` 都老老实实地进 callee 跑一遍 TLV thunk——**CSE 没有机会
发生**。

到这里 macOS 这边的真相清楚了：**Apple Clang 把 TLV thunk 的返回值当成"函数体内不变的
纯函数返回值"做了 CSE，跨过 `jump_context` 也照样复用——但 fiber 切换可能换了 OS 线
程，TLV 解析出来的"当前线程那块 TLS"已经不一样了。**

---

## 七、横向对比：写个最小例子，跑三种编译器

但是疑问还在：为什么 Linux 同样的代码、`noinline` 据说之前根本没有，怎么就没出过事？

把代码精简成最小可编译的形式：

```cpp
extern "C" [[gnu::returns_twice]] void jump_context(void**, void*, void*);
thread_local int* current_fiber;

static inline int* GetCurrent() { return current_fiber; }
static inline void SetCurrent(int* p) { current_fiber = p; }

int Test() {
    int* before = GetCurrent();
    jump_context(nullptr, nullptr, nullptr);
    SetCurrent(before);
    int* after = GetCurrent();
    return before == after ? 0 : 1;
}
```

然后用三种不同的工具链编：

```sh
clang++ -O2 -S -arch arm64 test.cc                          # Apple Clang，Mach-O TLV
clang++ -O2 -S --target=aarch64-linux-gnu test.cc           # Clang 重定向到 Linux ELF TLS
aarch64-elf-g++ -O2 -S test.cc                              # GCC，ELF TLS
```

三份汇编对照（这里只摘 `jump_context` 前后的关键片段，源码和编译命令见文末附录）：

### Apple Clang（macOS arm64，Mach-O TLV）

```asm
    adrp x0, _current_fiber@TLVPPAGE
    ldr  x0, [x0, _current_fiber@TLVPPAGEOFF]
    ldr  x8, [x0]                            ; 取 TLV 的 thunk 函数指针
①   blr  x8                                  ; ←── 关键①：调 thunk，解析槽地址
②   mov  x19, x0                             ; ←── 关键②：把槽地址塞进 callee-saved x19
    ldr  x20, [x0]                           ;       before = *槽
③   bl   _jump_context                       ; ←── 关键③：切换上下文
④   str  x20, [x19]                          ; ←── 关键④：*x19（还是旧地址）= before
    ret
```

### Clang（重定向到 aarch64-linux-gnu，ELF initial-exec）

```asm
①   mrs  x8, TPIDR_EL0                       ; ←── 关键①：读 thread pointer
    add  x8, x8, :tprel_hi12:current_fiber
②   add  x19, x8, :tprel_lo12_nc:current_fiber  ; ←── 关键②：槽地址进 callee-saved x19
    ldr  x20, [x19]                          ;       before = *槽
③   bl   jump_context                        ; ←── 关键③：切换上下文
④   str  x20, [x19]                          ; ←── 关键④：*x19（还是旧地址）= before
    ret
```

形状一模一样！只不过 macOS 上是 TLV thunk 解析出槽地址，Linux 上是 `mrs TPIDR_EL0`
加上链接器解析的偏移算出槽地址；两边都把结果存进 callee-saved `x19` 里、跨过
`jump_context` 继续用——**完全相同的 CSE bug**。

### GCC（aarch64 ELF）

```asm
①   mrs  x0, tpidr_el0                       ; ←── jump_context *之前* 解析一次
    add  x0, x0, #:tprel_hi12:.LANCHOR0, lsl #12
    add  x0, x0, #:tprel_lo12_nc:.LANCHOR0
    ldr  x0, [x0]
②   str  x0, [sp, 24]                        ; ←── 关键②：before 溢出到栈
                                             ;     而不是 callee-saved 寄存器
③   bl   jump_context                        ; ←── 关键③：切换上下文
④   mrs  x0, tpidr_el0                       ; ←── 关键④：jump_context 之后
                                             ;            *重新* 解析 thread pointer
    add  x0, x0, #:tprel_hi12:.LANCHOR0, lsl #12
    add  x0, x0, #:tprel_lo12_nc:.LANCHOR0   ; ←── 关键：重新算出槽地址
    ldr  x1, [sp, 24]
    str  x1, [x0]                            ;       *新槽地址 = before  ✓ 当前线程
    ret
```

**关键差别一句话**：

| 编译器                    | `jump_context` 后干了什么            |
| ------------------------- | ------------------------------------ |
| Apple Clang (macOS TLV)   | 重用调用前缓存在 `x19` 里的槽地址 ✗  |
| Clang (Linux ELF-TLS)     | 重用调用前缓存在 `x19` 里的槽地址 ✗  |
| GCC (Linux ELF-TLS)       | 重新 `mrs tpidr_el0` + 重算偏移 ✓    |

GCC 对 `[[gnu::returns_twice]]` 的处理就是更保守一点：跨 call 存活的值不放进 callee-
saved 寄存器、改放栈上（这样寄存器分配就不能复用了），同时 TLS 寻址重做一遍。Clang
只把"寄存器值可能不同"当成 spec 字面意思——但 `x19` 是 callee-saved，调用前后值"看
起来不变"，那就接着用。

到这一步，"为什么 Linux 没事"的真相浮出来了：

> **不是 Linux 比 macOS 安全，是 GCC 比 Clang 保守。** Linux 历史上是用 GCC 构建的，
> 所以这个 bug 一直没暴露。同样的 flare 代码要是在 Linux 上用 Clang 编，按这个最小
> 复现的结果，会出一模一样的事。

---

## 八、为什么 `returns_twice` 没救我们？

`returns_twice` 的语义是从 `setjmp`/`longjmp` 来的：

- 调用前所有寄存器被视为"将死"（caller 不能假设它们还有效）。
- 第二次返回时，局部变量的值可能被改过。

这条规约里没有一句话提到"调用之后可能在另一个 OS 线程上跑"。`setjmp` 不会换线程，所以
TLS 地址对 `setjmp` 来说**真的是函数体内不变的**——把它缓存在寄存器里是正确的优化。

Clang 严格按 spec 来：寄存器值可能不同 → 保守。但 TLS 寻址被视为"和线程绑定的、函数内
不变"的属性 → 可以 CSE。

GCC 给了我们更宽松的"假设最坏情况"的对待，但这只是 GCC 实现选择上的保守，不是 spec
保证。

**`returns_twice` 是为 setjmp 设计的，不是为 fiber 设计的。** 没有任何标准属性能说清楚
"这个调用可能换 OS 线程"。

---

## 九、为什么 `volatile` 和 memory barrier 都不管用？

第三、四节里那两次失败，根因是**完全一样**的。访问一个 `thread_local` 变量在编译器眼
里分成两步：

1. **解析槽地址**（macOS 上是 `blr x8` 调 TLV thunk；Linux 上是 `mrs TPIDR_EL0` +
   链接器偏移）。
2. **读/写槽**（普通的 `ldr`/`str`）。

`volatile thread_local` 只覆盖了第 2 步——逼着每次 `ldr`/`str` 都老实发出来，不能被
优化掉。但第 1 步——槽地址的解析——不在 `volatile` 的语义里，所以 Apple Clang 把第
1 步的结果缓存在 `x19`、第 2 步每次乖乖重读，结果一直 load 的还是同一个**陈旧**地
址。

`asm volatile("" ::: "memory")` 同理：它只说"内存里的内容可能被改了"，所以编译器会把
变量本身的最新值重新 load 一遍——但 `x19` 装的是个"系统寄存器 + 常量"算出来的地
址，不是从内存 load 的值。"memory" clobber 没有任何理由让一个不依赖内存的寄存器内容
失效。

一句话总结这一组失败：

> `volatile`、`asm("" ::: "memory")`、`returns_twice` ——**这三个机制覆盖的都是
> "值"，覆盖不到"地址"**。而我们要防的偏偏是"槽地址被跨 call 缓存"。

---

## 十、为什么 x86_64 完全没出过这个 bug？

x86_64 的 TLS 访问长这样：

```asm
mov %fs:offset, %rax
```

`%fs:` 是段覆盖，**段寄存器是每条指令隐式带上的**——CPU 加载 `%fs:offset` 时由操作系
统当前的 `fs` 段去算地址，没有"线程基址"这种东西被缓存在通用寄存器里。所以从根上就没
有可以 CSE 掉的"槽地址"，这一整个问题在 x86_64 上不存在。

AArch64 上 `mrs TPIDR_EL0` 读的是系统寄存器，但读出来的值会被编译器存进通用寄存器并
缓存。Mach-O TLV 更过分——整个解析是个函数调用，结果天然就是一个"值"，特别适合被
CSE。

**同一类 bug 在 x86_64 上不出现，是 ISA 选择无意中给我们兜了底。**

---

## 十一、修复：`noinline`，以及为什么是它

回到 flare 的代码：

```cpp
[[gnu::noinline]] FiberEntity* GetCurrentFiberEntity() noexcept { return current_fiber; }
[[gnu::noinline]] void SetCurrentFiberEntity(FiberEntity* p)    { current_fiber = p; }
```

关键是要看清楚 `noinline` 在这里到底"反掉"的是哪个优化。**它反掉的不是"对 TLS 的优
化"**——`current_fiber` 的读写本身是单条 `ldr`/`str`，没什么可优化的。它反掉的是另一
个更具体、更隐蔽的优化：

> 编译器看到一个函数体内多次访问同一个 `thread_local` 变量时，会把"解析当前线程
> 的 TLS base"这一步**只做一次**，结果存进 callee-saved 寄存器（比如 `x19`），后续
> 几次访问都从这个寄存器派生槽地址。

这个优化本身在一般情况下是合法且正确的——同一个函数体里，"当前线程"通常确实不会
变。但是 fiber 切换偏偏让它变了。

`noinline` 让这个"函数体内只解析一次 TLS base"的优化**前提**不再成立。被加上
`noinline` 之后，`GetCurrentFiberEntity` 就是一个完整的、独立的函数：它的"函数体内"
只有一次 TLS 访问，所以"只解析一次"在它内部仍然合法（实际上也只解析一次）；但
caller `Resume()` 看到的只是一个不透明的 `bl GetCurrentFiberEntity` 调用——按 ABI 规
约，外部函数的返回值不能在两次调用之间被假设相等，所以"在 Resume 里只解析一次"这件
事根本不可能被推出来。优化失效，bug 自然不见。

代价是每次访问多一次函数调用——几条指令（push 帧、`bl`、`ret`）——比起一次 fiber
切换的开销（保存恢复整个寄存器组、可能换 OS 线程），是完全可以忽略的。

而且这个修复对 Clang 和 GCC 都管用，不依赖任何特定编译器的"自觉行为"。**这才是这件事
最重要的属性**——bug 是编译器之间的差异造成的，修复就不应该再依赖某个编译器的具体行
为。

---

## 十二、如果只是把它挪到 `.cc` 文件呢？—— LTO 的陷阱

读到这里很自然会有个想法：既然 `noinline` 起作用是因为 ABI 调用边界天然抗 CSE，那
**把 `GetCurrentFiberEntity` 的定义从 header 挪到 `.cc` 文件里**——让它变成跨 TU 的
外部符号——是不是就够了？这样普通构建里 caller 看不见 callee 的实现，必然要发 `bl`
指令，CSE 同样被切断。

普通构建下，**是的，确实够**。但是开了 LTO 之后，**bug 会回来**。

LTO（包括 Apple 工具链里实际上已经默认开的 ThinLTO）让链接器拿到的不是机器码，而是
IR——LLVM bitcode 或 GCC GIMPLE。链接阶段可以**跨 TU 内联**，而像
`GetCurrentFiberEntity` 这种只有几行的纯访问器，是 LTO 内联启发式最显然的目标，几乎
必然被吃掉。一旦在链接期内联进 `Resume()`，反汇编就回到了第六节那个出 bug 的形状——
`x19` 又被重复使用。

`[[gnu::noinline]]` 的关键属性正在这里：它**把"绝对不许内联"这个约束钉死在函数定义
里**，跟着函数一起走到 LTO 链接器、走到 ThinLTO、走到未来任何更激进的优化配置里。
"挪到 `.cc`"是利用了**当下构建模式的 TU 边界**，而 `noinline` 是利用了 **attribute
在 IR 里的持久语义**——两者的鲁棒性不在一个量级上。

三种位置 × 是否开 LTO 的安全性：

| 定义位置                        | 不开 LTO | 开 LTO / ThinLTO |
| ------------------------------- | -------- | ---------------- |
| Header（隐式 `inline`）         | ✗ bug    | ✗ bug            |
| `.cc` 文件（跨 TU）             | ✓ 安全   | **✗ bug 回来**   |
| `[[gnu::noinline]]`（无论位置） | ✓ 安全   | ✓ 安全           |

如果想再加一层保险，还有 `[[gnu::noipa]]`（"no interprocedural analysis"，连过程间分
析都不许做）——比 `noinline` 更强。flare 这个场景实际上单 `noinline` 已经够稳：我们
要防的就是"caller 在 callee 调用两侧共享寄存器内容"，函数边界本身已经把这条切断了，
不需要再阻断 IPA。但如果担心未来某天 IPA 推断出"这个函数总是返回 thread-local"然后把
寄存器关联起来推理，加 `noipa` 可以多挡一道。

教训：**ABI 边界天然抗 CSE 这条性质成立——前提是 ABI 边界确实存在。** LTO 时代，TU
边界已经不再天然就是 ABI 边界，需要用 attribute 显式声明才是。

---

## 十三、收获

整理一下这次踩坑的几个教训：

1. **编译器属性按 spec 解读，不按"我以为它的意思"解读。** `returns_twice` 看着像万金
   油，实际上它的语义只覆盖 `setjmp` 那一类同线程的"第二次返回"，根本没有覆盖"线程
   可能换了"这件事。`volatile`、`asm("" ::: "memory")` 同理——它们说的是"重新 load
   /store 这个值"，不是"重新解析 thread-local 的寻址"。

2. **"覆盖值"和"覆盖地址"是两件事。** 我们这次试过的所有"软"机制（`returns_twice`、
   `volatile`、memory barrier）保证的都是"调用前后值可能不同"，而 fiber 真正要防的是
   "调用前后地址可能不同"——后者目前没有任何标准 C/C++ 机制能直接表达。

3. **"在 X 上跑得好"经常其实是"X 平台默认的编译器跑得好"。** Linux + GCC 工作，不代
   表 Linux 安全；Linux + Clang 会重现 macOS 上的 bug。把"Linux"和"GCC"分开看，结论
   会立刻清楚。

4. **猜不动的时候，反汇编是终极裁判。** 写一段最小复现，跑三个编译器，把 `_Z4Testv:`
   到 `ret` 之间那 20 行汇编贴出来——比看一千行优化器源码都直接。这次三个失败的猜想
   全是被反汇编一锤定音的。

5. **ABI 边界天然抗 CSE——前提是边界还在。** 当你想阻止某种"跨某次调用的优化"但又找
   不到合适的编译器属性时，把那个访问藏到一个 `noinline` 的小函数后面，是个非常便宜、
   非常通用的招。注意：**单纯"把定义挪到 `.cc` 里"在 LTO（含 ThinLTO）下不够**，
   attribute 才是把约束钉死在 IR 里的那一刀。inline asm clobber 可以模拟这件事，但
   需要列出所有 GP 寄存器才完整——还不如直接借 ABI。

6. **任何"调用之后可能换 OS 线程"的原语都有这个隐患。** 用户态线程库
   （fiber、user-mode 协程）、`ucontext` 上面的封装、跨线程 work-stealing 调度的入口
   函数等等。C/C++ 没有标准属性能告诉编译器这件事，所以底层切换原语两侧的 thread_local
   访问都得防一手。

---

## 附录：最小复现源码

把下面两个文件保存到本地，照后面的命令分别编一下，就能复现正文里那三段反汇编。

### `test.cc` — 原始版（不加 barrier）

```cpp
// 模拟 flare 里 jump_context + thread_local 的最小代码骨架：
//   - 一个 thread_local 指针（对应 current_fiber）
//   - 一个 returns_twice 的外部函数（对应 jump_context）
//   - 读、调切换、写、再读，验证前后是否一致
// 用 -O2 -S 看 Test() 函数体，重点观察 jump_context 前后的 TLS 槽地址是否被复用。

extern "C" [[gnu::returns_twice]] void jump_context(void** a, void* b, void* c);

thread_local int* current_fiber;

static inline int* GetCurrent()      { return current_fiber; }
static inline void SetCurrent(int* p) { current_fiber = p; }

int Test() {
    int* before = GetCurrent();
    jump_context(nullptr, nullptr, nullptr);
    SetCurrent(before);
    int* after = GetCurrent();
    return before == after ? 0 : 1;
}
```

### `test_barrier.cc` — 验证 `asm volatile memory` 那一节

```cpp
// 跟 test.cc 完全一样，只在 jump_context 之后多了一行编译器内存屏障。
// 用来证明这条 barrier 并不会让 Clang 重新解析 TLS 槽地址。

extern "C" [[gnu::returns_twice]] void jump_context(void** a, void* b, void* c);

thread_local int* current_fiber;

static inline int* GetCurrent()      { return current_fiber; }
static inline void SetCurrent(int* p) { current_fiber = p; }

int Test() {
    int* before = GetCurrent();
    jump_context(nullptr, nullptr, nullptr);
    asm volatile("" ::: "memory");   // <-- 候选 fix
    SetCurrent(before);
    int* after = GetCurrent();
    return before == after ? 0 : 1;
}
```

### 三种工具链的编译命令

```sh
# Apple Clang，macOS arm64，Mach-O TLV
clang++ -O2 -S -arch arm64 -o macos.s test.cc

# Apple Clang 重定向到 aarch64-linux-gnu（不需要 sysroot，只输出汇编）
clang++ --target=aarch64-linux-gnu -O2 -S -o linux_clang.s test.cc

# aarch64 ELF GCC（macOS 上 brew install aarch64-elf-gcc）
aarch64-elf-g++ -O2 -S -fno-exceptions -fno-rtti -o gcc.s test.cc
```

每份 `.s` 文件里直接找 `_Z4Testv:`（`Test()` 函数的 mangled 符号），到下一个 `ret`
之间那二十来行就是正文里贴的全部内容。
