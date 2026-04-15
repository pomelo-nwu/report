# 极限挑战：那些让人怀疑"这真的能做？"的任务

> TBench 89 个任务中有一批让人第一反应是"不可能"的任务。它们不是普通的工程题——它们是 Agent 能力的真正边界。逐一拆解这些任务，你会发现每个"不可能"背后都有一个令人惊叹的解题策略。

---

## 开篇：什么是"极限"？

我先定义一下"极限任务"的标准：不是简单的"难度标签为 hard"——很多 hard 任务只是时间长，策略仍然直觉。真正的极限任务满足以下条件之一：

1. **直觉告诉你不可能**——第一反应是"这怎么能做？"
2. **最优解法不是直觉告诉你的那个**——需要跳出思维定式
3. **expert/junior 比率极高**——专业知识差距指数性

我选了 6 个这样的任务，按"震撼度"排序。

---

## 🏆 1. circuit-fibsqrt：在门电路层构建 CPU

**难度**：hard | **比率**：2.5x (960min / 2400min)

### 任务描述

创建 `/app/gates.txt`（<32000 行），每行是逻辑门表达式（`outX = outY & outZ`、`outX = outY ^ outZ` 等）。前 32 行由输入的二进制表示初始化，经过 32000 步仿真后，最后 32 行转为整数输出。要求实现 **fib(isqrt(N)) mod 2³²**。

给出两个验证值：`sim(208)=377`，`sim(20000)=1407432322`。

### 为什么"不可能"？

用逻辑门实现 Fibonacci + 整数平方根？这听起来像计算机科学课上的理论练习——**用什么门电路能直接算 Fibonacci？**

答案：不能。门电路只能做 AND/OR/XOR/NOT——连加法都需要多步组合。Fibonacci 依赖加法和迭代，isqrt 依赖减法和比较。直觉上你会试图直接用门电路实现这些运算，然后发现 32000 步根本不够嵌套循环。

### Oracle 的惊人策略

Oracle 不是直接用门电路实现数学函数，而是**先在 Python 中构建完整 CPU 的 Signal/Bits 抽象层，然后导出为 gates.txt**：

```
逻辑门 (AND/XOR/NOT/MUX)
    ↓ 组合
DFF (D触发器/主从两级半锁存器)
    ↓ 组合
ALU (16 种操作：加减/比较/移位/AND/OR/XOR/NOT...)
    ↓ 组合
寄存器文件 (16×32bit，MUX + DFF 实现)
    ↓ 组合
ROM (45 条 32 位指令，MUX 树地址查找)
    ↓ 组合
完整 CPU (8周期时钟：取指→解码→执行→写回)
    ↓ 编程
fib(isqrt(N)) mod 2³² (微代码硬编码)
```

这是**在一个虚拟 CPU 上运行虚拟程序来计算真实数学函数**。300 行 Python 构建了一台计算机，然后在计算机上编程。

### 评分的精确性

test_outputs.py 编译 `/tests/sim.c`（gcc -O3），对 28 个测试值（1, 4, 8, 12, 41, 42, 107, 220, 209, 366 及其平方和平方±1）逐一运行 `sim N`，比对 `fib(isqrt(N)) mod 2³²` 的预期值。28 个精确值，任何一个不匹配 = 0 分。

### 可迁移洞察

> **当你面对一个看似不可能的约束时，不要试图直接解决，而是先构建一个足够强大的基础，然后在基础上解决。**

这不是编程技巧——是一种**工程思维的范式**：从微观优化转向宏观架构。一个只会"用门电路实现加法器"的 Agent 会陷入局部死循环；一个能识别"我需要先建 CPU"的 Agent 才能找到出路。

**对你的项目意味着什么**：当你面对极端约束时，第一步不是优化当前方案，而是问自己——"我是不是需要一个更强大的抽象层？"

---

## 🏆 2. regex-chess：用纯正则表达式实现象棋引擎

**难度**：hard | **比率**：3.3x (1440min / 4800min)

### 任务描述

写 `/app/re.json`，包含 `[regex, replacement]` 对列表。通过 Python `re.sub` 依次执行所有替换后，应从 FEN 位置生成所有合法下一步棋的位置列表。必须正确处理王车易位、升变（只Queen）、过路兵。re.json < 100,000 对，< 10MB。

### 为什么"不可能"？

**正则表达式不具备状态追踪能力。** 它不能"记住"棋盘上有哪些棋子、哪些位置被占用、哪些走法合法。国际象棋的走法生成需要完整的棋盘状态——这是一个理论上不可能用纯 regex 解决的问题。

### Oracle：数据而非逻辑

Oracle 的 solve.sh 是一个**巨大的 gzip+base64 编码数据块**，解码后直接写入 `/app/re.json`。没有代码逻辑——只有模式匹配规则。

这揭示了一种解题思路：**纯 regex 确实不能追踪状态，但如果把所有可能的状态都"展开"成不同的 regex 模式，问题就变成了一个巨大的模式匹配问题。** 代价是规则数量极其庞大（<100,000 对的限制），但理论上可行。

### 评分的三局经典棋局

test_outputs.py 使用 `python-chess` 库生成参考合法走法，对 3 个经典棋局的所有位置验证：

- Anderssen's Immortal Game (1851)
- Fischer's Game of the Century (1956)
- Naroditsky vs Ivanchuk (2024 World Blitz)

任何一个位置的走法生成有误 = 0 分。

### 可迁移洞察

> **有时候"不可能"只是意味着你需要一种非传统的表示方式。**

这也揭示了 Agent 能力的一个有趣维度：能否在"理论不可行"的约束下找到"实践可行"的解法？regex-chess 的 expert 估计 1440 分钟（24 小时）——全 benchmark 最长之一——说明即便对专家来说，这也是极端挑战。

---

## 🏆 3. feal-differential-cryptanalysis：差分密码分析攻击

**难度**：hard | **比率**：40x (480min / 19200min) | **全 benchmark 最高比率之一**

### 任务描述

对 `/app/feal.py` 中的 FEAL-like 加密函数实施选择明文攻击，恢复 `key[5]` 的 uint32 值。写入 `/app/attack.py`，实现 `attack(encrypt_fn)` 函数，运行时间 < 30 秒。

提示信息：6 轮密钥各由 16 位种子派生（`key[i] = seed * 1234567 & 0xFFFFFFFF`），暴力搜索整个密钥空间不可行，但差分攻击可行。

### 为什么"不可能"？

这不是"写一段代码"——这是**真正的密码分析**。Agent 需要理解差分密码分析的理论、FEAL 的已知弱点、选择明文攻击的实现方式。没有密码学背景，19200 分钟（320 小时）的估计意味着"可能根本做不出来"。

### Oracle 的攻击策略

```python
# 差分特征：0x8080000080800000 → 0x02000000
# 这是 FEAL 的经典已知弱点

# 对每个 16 位种子候选（0-0xFFFF）：
# 1. 随机选择 plaintext p
# 2. 加密 p 和 p^b（差分）
# 3. 计算 (c1l ^ c2l) == 0x02000000
# 4. 保留满足条件的候选
# 5. 迭代 10 轮缩小候选集
```

关键：seed 派生公式 `(seed * 1234567) & 0xFFFFFFFF` 使得只有 65536 个种子候选（16 位），而差分攻击可以高效缩小候选集。

### 评分：编译 + 调用

test_outputs.py 先编译 C 实现的 FEAL（`python setup.py build_ext --inplace`），然后调用 `attack.attack(feal_in_c.encrypt)`，验证返回值等于 `feal_in_c.get_keys()[5]`。

### 姊妹任务：feal-linear-cryptanalysis

仓库中还有 **feal-linear-cryptanalysis**，同样有 40x 比率（expert 960, junior 19200）。差分和线性密码分析是 FEAL 的两种经典攻击方式——TBench 把两种都收录了，形成互补的评测维度。

### 可迁移洞察

> **专业知识的价值不是线性的，而是指数性的。** 40x 比率意味着懂的人 8 小时搞定，不懂的人可能 320 小时仍失败。

对 Agent 开发者的启示：**领域知识注入**（通过 RAG、文档检索等方式）对极端专业任务可能是关键能力。一个不懂密码分析的 Agent，再强的推理能力也帮不了它——它需要"知道"差分特征 `0x8080000080800000 → 0x02000000` 这种事实。

---

## 🏆 4. code-from-image：识别"捷径"的元认知能力

**难度**：medium | **比率**：4x (30min / 120min)

### 任务描述

从 `/app/code.png` 的伪代码图片提取逻辑，实现并产生相同输出，写入 `/app/output.txt`。提示正确答案以 `bee26a` 开头。

### 为什么"极限"？

它不是 hard——但它的解法是最"反直觉"的。instruction.md 明确说："Default OCR tools like tesseract/easyocr will fail to extract the exact code." 甚至补充："SmolVLM-Instruct (2B) can almost extract the code"——暗示小模型几乎能做但不够精确。

直觉解法：OCR → 读代码 → 执行 → 输出。但 OCR 不精确，怎么办？

### Oracle 的元认知捷径

Oracle 的注释揭示了设计意图："The code.png is generated intentionally so default OCR tools will fail." 解法完全跳过了 OCR：

```python
SALT = b"0000TBENCH-SALT"
img_bytes = read_file_bytes("/app/code.png")
h0 = sha256(img_bytes)
H = sha256(h0, h0[:10], SALT)
print(H.hex())  # 输出 bee26a133f103b9ecda444c70ec22cafef6e31a3de7af6d047974dc90ce3defe
```

图片文件本身的 SHA256 哈希就是输出的确定性来源！**不需要理解代码内容，只需要理解"输出是由图片确定的"这个元信息。**

### 评分：精确哈希匹配

```python
expected = "bee26a133f103b9ecda444c70ec22cafef6e31a3de7af6d047974dc90ce3defe"
assert output.strip() == expected
```

### 可迁移洞察

> **有时候最聪明的解法不是"按规矩做"，而是"理解问题的本质后发现捷径"。**

规矩做法是 OCR→执行→输出；但问题的本质是"产生特定输出"——而图片本身就是这个输出的确定性来源。

**这测试的不是 Agent 的编码能力，而是元认知能力**——能否识别何时应该跳出常规思路。一个只会"按规矩做"的 Agent 会花数小时调 OCR 参数；一个能识别捷径的 Agent 30 分钟就搞定。

---

## 🏆 5. install-windows-3.11：30 年前的操作系统与版本兼容性陷阱

**难度**：hard | **比率**：2x (300min / 600min)

### 任务描述

使用 QEMU 运行 Windows 3.11 for Workgroups 虚拟机（镜像 `/app/isos/win311.img`），配置 VNC (port 5901) + nginx (port 80) web 接口，QEMU monitor socket 支持程序化键盘输入。

### 为什么"极限"？

这不是"安装软件"——这是**跨越 30 年的版本兼容性迷宫**。Windows 3.11 发布于 1993 年，QEMU 5.2.0 发布于 2020 年。核心难点：

- **QEMU 版本必须精确**：Windows 3.11 只在 QEMU 5.2.0 上稳定运行，新版不行
- instruction.md 给出了版本要求但没有解释为什么——Agent 需要自己搜索发现兼容性问题
- 交互式验证：测试通过 QEMU monitor socket 发送键盘输入

### Oracle：从源码编译 QEMU 5.2.0

Oracle 的 solve.sh 约 150 行，核心操作是**从源码编译 QEMU 5.2.0**——因为容器中没有预装的精确版本。这是一个系统管理任务中最典型的"隐性知识"场景：文档不会告诉你"Windows 3.11 只在 QEMU 5.2.0 上运行"，你需要从论坛、Stack Overflow 等来源发现这个事实。

### 评分：最创新的验证方式

install-windows-3.11 的评分是整个 TBench 中最复杂的——约 200 行 Python，包含 5 层验证：

1. **网络状态**：VNC port 5901 + nginx port 80 可访问
2. **QEMU 参数**：验证 snapshot 模式、镜像路径、VGA 配置、内存配置
3. **镜像 SHA256**：验证 QEMU 镜像哈希匹配已知哈希集（2 个已知哈希）
4. **核心文件 SHA256**：用 mtools 从镜像提取 7 个 Windows 核心文件（WIN.COM, PROGMAN.EXE, WINFILE.EXE, CONTROL.EXE, KRNL386.EXE, USER.EXE, GDI.EXE），逐一哈希比对，至少 5 个匹配
5. **键盘交互验证**：通过 QEMU monitor 发送键盘命令（F1, Alt+Tab, F10, Alt+F4, Ctrl+Esc），用 vncsnapshot 截图 + cv2 计算像素差异 ≥ 10%

第 5 层是最创新的部分——**用计算机视觉验证虚拟机的交互响应**。这不是"检查文件是否存在"，而是验证"键盘输入真的影响了屏幕显示"。

### 可迁移洞察

> **版本兼容性是系统管理的"隐性知识"**——它不在文档里，只存在于论坛帖子和失败经验中。

对 Agent 的启示：Web 搜索能力在系统管理类任务中至关重要。一个没有搜索能力的 Agent 只能用"试试最新版本"的策略，而一个能搜索的 Agent 可以发现"QEMU 5.2.0"这个关键事实。

---

## 🏆 6. schemelike-metacircular-eval：能解释自身的求值器

**难度**：medium | **比率**：8.1x (300min / 2440min)

### 任务描述

编写 `eval.scm`——一个元循环求值器，能解释 scheme-like 语言，且**能解释自身**。

验证方式是三层元循环：
```
echo '(+ 7 8)' | python3 interp.py test/calculator.scm      = 15  ← 一层
echo -e 'test/calculator.scm\n(+ 7 8)' | python3 interp.py eval.scm = 15 ← 二层
echo -e 'eval.scm\ntest/calculator.scm\n(+ 7 8)' | python3 interp.py eval.scm = 15 ← 三层
```

### 为什么"极限"？

8.1x 的比率说明：理解了元循环求值器的概念后实现不难（300 分钟），但没理解概念的人几乎无从下手（2440 分钟）。这是 **SICP** 的经典挑战——自指性是计算理论的基石。

### Oracle 的实现

约 200 行 Scheme 实现了完整求值器：

| 功能 | 实现 |
|------|------|
| 环境操作 | make-frame, extend-env, lookup, set-var!, define-var! |
| eval | 数字/字符串/布尔/符号/quote/if/define/set!/lambda/begin/cond/let |
| apply | 原始操作 vs 复合过程 |
| 原始操作 | +,-,*,/,=,<,>,<=,>=,cons,car,cdr,null?,pair?,eq?,equal?,not,symbol?,set-car!,set-cdr!,number?,string? |
| 文件 I/O | fopen/fclose/fread/fgetchr/fputchr/fdisplay,read,display,newline,putchr |

三层元循环验证确保了求值器真正具备自指能力，而非仅仅是一个"凑巧能跑"的实现。

### 可迁移洞察

> **自指性（self-reference）是计算理论的基石，也是工程系统的终极测试。** 一个能解释自身的求值器是"计算完备性"的终极证明。

为什么 difficulty 标签是 medium 而非 hard？因为"一旦理解了概念，实现并不特别困难"——8.1x 的比率不是来自实现难度，而是来自**概念门槛**。这对 Agent 的启示是：概念理解比编码能力更基础——一个不理解"元循环求值器是什么"的 Agent，再强的编码能力也帮不了它。

---

## 跨任务共性：极限任务揭示了什么？

### 共性一："捷径发现"能力

| 任务 | 捷径 | 直觉解法 |
|------|------|---------|
| circuit-fibsqrt | 先建 CPU 再编程 | 直接用门电路实现 |
| code-from-image | SHA256 哈希 | OCR 提取代码 |
| install-windows-3.11 | 编译 QEMU 5.2.0 | 用最新 QEMU |

最优解法往往不是直觉告诉你的那个。能否发现捷径，是区分优秀 Agent 和普通 Agent 的关键能力。

### 共性二：专业知识的价值是指数性的

```
Expert/Junior 比率梯度：
  1x  ← 任何工程师都能尝试
  2-3x ← 需要一些经验
  8x  ← 需要概念理解
  10x ← 需要领域背景
  24x ← 需要特定领域知识
  40x ← 需要深度专业知识
```

比率越高，领域知识注入的价值越大。通用推理能力在高比率任务上帮不了太多。

### 共性三：概念理解 > 编码能力

circuit-fibsqrt 需要理解"先建基础设施再解决问题"，schemelike-metacircular-eval 需要理解"元循环求值器是什么"，code-from-image 需要理解"输出是由图片确定的"。这些都不是编码问题——是**概念理解问题**。

---

## Key Takeaways

1. 极限任务的定义不是"难度标签为 hard"，而是"直觉告诉你不可能"或"最优解法反直觉"
2. circuit-fibsqrt 的"先建 CPU 再编程"策略揭示了一种工程思维范式：面对极端约束，先建基础设施再解决
3. 40x 的 expert/junior 比率意味着专业知识价值指数性——领域知识注入对极端专业任务可能是关键能力
4. code-from-image 的 SHA256 捷径揭示了元认知能力的重要性：理解问题本质比按规矩做更重要
5. 概念理解 > 编码能力——不理解"元循环求值器是什么"的 Agent，再强的编码能力也帮不了它