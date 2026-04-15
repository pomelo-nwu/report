# 从 task.toml 到 reward.txt：一个任务的完整工程链路

> TBench 的工程架构不是"几道题放在 Docker 里"那么简单。从配置到评分，每一个环节都经过精密设计——这背后是"评测可信度"的硬性工程要求。

---

## 开篇：为什么工程链路比评测本身更重要？

你可能觉得评测基准的核心是"题目设计"。但 TBench 的情况反直觉：**题目只是冰山一角，真正决定评测可信度的是从定义到验证的整条工程链路。**

原因很简单——如果评分不可信，再好的题目也是废纸。一个 Agent 通过了某道题，你怎么知道它真的做对了，而不是投机取巧？你怎么知道所有 Agent 在相同环境下运行？你怎么知道评分数据没有泄漏到训练集？

TBench 用 5 个标准组件构建了一条完整的工程链路。让我们逐层拆解。

---

## 第一层：task.toml——任务的宪法

每个任务的核心配置是 `task.toml`，版本号统一为 `"1.0"`。这是整个评测链路的起点——所有超时、资源、难度标签都在这里定义。

```toml
# feal-differential-cryptanalysis/task.toml（精确摘录）
version = "1.0"

[metadata]
author_name = "Nicholas Carlini"
difficulty = "hard"
category = "mathematics"
tags = ["cryptanalysis", "feal"]

expert_time_estimate_min = 480.0
junior_time_estimate_min = 19200.0

[verifier]
timeout_sec = 1800.0

[agent]
timeout_sec = 1800.0

[environment]
build_timeout_sec = 600.0
docker_image = "alexgshaw/feal-differential-cryptanalysis:20251031"
cpus = 1
memory = "2G"
storage = "10G"
```

### 双轨时间估计：人类参照系

`expert_time_estimate_min` 和 `junior_time_estimate_min` 是 TBench 最精妙的设计之一。它不是简单的"预计耗时"，而是为 Agent 表现提供**人类参照坐标系**。

极端比率最能说明问题：

| 任务 | Expert (min) | Junior (min) | 比率 | 含义 |
|------|-------------|-------------|------|------|
| feal-differential | 480 | 19200 | **40x** | 没有密码分析知识几乎不可能 |
| custom-memory-heap-crash | 30 | 1200 | **40x** | 内存调试门槛极高 |
| break-filter-js-from-html | 20 | 480 | **24x** | 需要特定 Web 安全知识 |
| prove-plus-comm | 5 | 120 | **24x** | 形式化证明有专业门槛 |
| overfull-hbox | 60 | 60 | **1x** | 任何工程师都能尝试 |

40x 的比率意味着：一个懂密码分析的人 8 小时能搞定，一个不懂的人可能需要 320 小时——**而且很可能仍然失败**。专业知识的价值不是线性的，而是指数性的。

### 资源配置：克制与精确

89 个任务中：

| 资源 | 众数 | 极值 | 分布 |
|------|------|------|------|
| CPUs | 1 | 4 | 1核: 84个, 2核: 3个, 4核: 2个 |
| Memory | 2G | 8G | 2G: 68个, 4G: 17个, 8G: 2个 |
| Timeout | 900s | 12000s | 众数 900s, 编译类可达 3.3 小时 |

**关键发现：资源需求与难度弱相关。** 高资源任务不是因为"难"，而是因为"大"。mcmc-sampling-stan 需要 4 核 8G 不是因为它难（difficulty: medium），而是因为 Stan 采样本身就吃资源。这确保了评测区分度来自能力而非硬件。

---

## 第二层：instruction.md——Agent 看到的唯一信息

instruction.md 是 Agent 在容器内接收到的任务描述。它的设计有一个反直觉的特征：**不人为统一风格**。

看三个极端对比：

**fix-git**（极简）：

```markdown
The git repo at /app has a merge conflict in the main branch.
Resolve the merge conflicts so that `make test` passes.
```

只说了目标，零提示。一个有 git 经验的 Agent 立刻知道怎么做；一个没有的 Agent 会四处搜索。

**feal-differential-cryptanalysis**（学术式）：

```markdown
You are given a FEAL-like encryption function in /app/feal.py.
Your task is to implement a chosen-plaintext attack...
Each 6-round key is derived from a 16-bit seed: key[i] = seed * 1234567 & 0xFFFFFFFF...
The differential characteristic 0x8080000080800000 -> 0x02000000...
```

描述了算法原理和约束——但没有说"你应该这样做"，而是给了原始信息让 Agent 自己推理。

**code-from-image**（带陷阱）：

```markdown
The image at /app/code.png contains pseudocode...
Default OCR tools like tesseract/easyocr will fail to extract the exact code.
The correct answer starts with bee26a...
```

明确说"默认 OCR 会失败"——这既是挑战也是暗示。Agent 需要识别到"既然 OCR 不行，也许根本不需要 OCR"。

**设计哲学**：真实世界中的任务指令千差万差——有的是一句话的需求，有的是学术论文级别的技术规格，有的是故意不完整的文档。TBench 不试图统一指令风格，因为这本身就是对 Agent "信息补全能力"的测试。

---

## 第三层：Dockerfile——可控环境的三重策略

### 基础镜像选择

TBench 的基础镜像选择遵循一条清晰的逻辑线：

```
python:3.13-slim-bookworm  ──── 60%+ 的任务（最小化 Python 环境）
ubuntu:24.04               ──── 系统级工具任务（编译、数据库、虚拟化）
debian:bullseye-slim       ──── QEMU 任务（历史版本兼容）
```

选择逻辑：需要纯 Python → slim-bookworm（最小体积）；需要 apt 安装系统工具 → ubuntu:24.04；需要历史版本兼容 → debian:bullseye。这不是随意选择，而是**最小化原则**——只用必要的环境，减少意外干扰。

### 预构建镜像：一致性锁

所有 89 个任务的 Docker 镜像都预构建并托管在 Docker Hub：

```toml
docker_image = "alexgshaw/task-name:20251031"
```

标签 `:20251031` 对应 2025 年 10 月 31 日——这是所有环境的同一天冻结版本。这意味着：

- ✅ 所有评测者使用完全相同的环境
- ✅ 不需要本地构建，直接拉取
- ⚠️ 如果某个基础镜像有安全漏洞，所有任务都受影响

**一致性优先于灵活性**——这是一个明确的工程取舍。

### 防泄露：先破坏再让 Agent 修复

安全类任务的 Dockerfile 有一种精妙的设计模式——**故意破坏答案**：

```dockerfile
# chess-best-move: 删除生成工具，防止 Agent 从环境中找到答案
RUN rm make.py

# fix-code-vulnerability: 用 sed 删除修复代码，让 Agent 自己修复
RUN sed -i 's/...vulnerable code.../...broken code.../' bottle.py
```

这确保了 Agent 必须真正理解漏洞才能修复，而不是从环境中复制答案。一个更深层的设计意图：**如果容器里有答案，Agent 可能不需要"理解"就能"完成"**——破坏答案是为了强制理解。

---

## 第四层：solve.sh——Oracle 的多样性与深度

Oracle 解决方案（solve.sh）是每个任务的"标准答案"。它们的复杂度跨度令人震惊：

```
极简 ←──────────────────────────────────────→ 极复杂

fix-git     openssl     db-wal     llm-batching    circuit      regex-chess
(3行)       (5行)       (50行)     (300行Python)   (300行Python) (巨型gzip+base64)
```

### 最震撼的 Oracle：circuit-fibsqrt

这个 Oracle 不是简单的脚本，而是一个**在 Python 中构建完整 CPU 的 Signal/Bits 抽象层，然后导出为 gates.txt** 的壮举：

```python
# solve.sh 中 CPU 架构的精确规格（从代码提取）
NUM_REGS = 16       # 16 个 32 位寄存器
REG_DEPTH = 32      # 每个寄存器 32 位深度
ALU_OPS = 16        # ALU 支持 16 种操作
CLOCK_PERIOD = 8    # 8 呯期时钟
ROM_SIZE = 45        # ROM 存储 45 条 32 位指令
```

CPU 的 ALU 操作包括：add, sub, AND, OR, XOR, NOT, const, shift-left, shift-right, any_set, const_input, pass-through（5个NOP）。微代码硬编码了计算 isqrt + fib 的程序。

**这是一个在虚拟 CPU 上运行虚拟程序来计算真实数学函数的系统**——Oracle 用 300 行 Python 构建了一台计算机，然后在计算机上编程。

### 最奇葩的 Oracle：regex-chess

这个 Oracle 不是代码，而是**压缩数据**。整个 solve.sh 是一个巨大的 gzip+base64 编码块，解码后直接写入 `/app/re.json`。因为用纯 regex 实现象棋走法生成，代码本身不是"逻辑"而是"模式匹配规则"——所以 Oracle 选择了数据压缩而非源代码。

---

## 第五层：test.sh → test_outputs.py → reward.txt——评分链路

### 统一的评分入口模板

89 个任务的 test.sh 几乎完全相同——一个高度标准化的模板：

```bash
# 所有任务的 test.sh 统一模板（只差 pytest 附加包）
apt-get update && apt-get install -y curl gcc
curl -LsSf https://astral.sh/uv/0.9.5/install.sh | sh
source $HOME/.local/bin/env
uvx -p 3.13 -w pytest==8.4.1 -w pytest-json-ctrf==0.3.5 [附加包] \
  pytest --ctrf /logs/verifier/ctrf.json /tests/test_outputs.py -rA
if [ $? -eq 0 ]; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

**关键设计决策**：

- **uvx 而非 pip**：每次验证都创建全新 Python 环境（`uvx -p 3.13`），避免环境污染
- **pytest-json-ctrf**：结构化测试报告（CTRF = Consistent Test Result Format），方便自动化分析
- **reward.txt**：二值文件，简单到不可能被误解读

这个模板的一致性说明：**评分框架是高度标准化的工程系统，而非每个任务自行发明**。

### 九种评分模式

通过分析多个任务的 `test_outputs.py`，我归纳出九种评分模式：

| 类型 | 名称 | 示例任务 | 验证方式 |
|------|------|---------|---------|
| A | 精确匹配 | count-dataset-tokens | `assert "79586" in output` |
| B | 哈希比对 | fix-git | MD5 哈希比较文件内容 |
| C | JSON 结构验证 | sqlite-db-truncate | 检查字段值、排序、无重复 |
| D | 数值阈值 | caffe-cifar-10 | `assert accuracy > 0.45` |
| E | 文件属性验证 | openssl-selfsigned-cert | 检查 key 位数、CN 名称、权限 |
| F | 编译执行比对 | circuit-fibsqrt | gcc 编译 sim.c，运行比对 28 个值 |
| G | IoU/几何比对 | sam-cell-seg | IoU ≥ 0.5 |
| H | API 级验证 | feal-differential | import + 调用函数验证返回值 |
| I | 多阶段组合 | fix-code-vulnerability | 原仓库 pytest + 自定义 pytest 都通过 |

九种模式的跨度从"检查字符串"到"编译运行再比对"——这确保了不同类型任务的评分精度与任务本身的复杂度匹配。

### 评分防作弊的四种武器

1. **SHA256 输入验证**（install-windows-3.11）：
   ```python
   known_hashes = {
       "58fb76014dccf13ec048e894ca1e44773eee940ccbc71c97a2ab3d07285da553",
       "63544885d8c574a55a7d01d122ac317e39202473c14b287c79caa68a72ea0740",
   }
   assert actual_hash in known_hashes
   ```
   防止 Agent 修改输入数据来适配自己的输出。

2. **禁止特定函数**（torch-pipeline-parallelism）：
   instruction.md 明确禁止 `register_forward_hook` 和 `register_full_backward_hook`——防止 Agent 用 hook 直接提取参考答案。

3. **像素差异验证**（install-windows-3.11）：
   ```python
   # 通过 QEMU monitor 发送键盘命令，用 vncsnapshot 截图
   # 用 cv2 计算像素差异，要求差异 ≥ 10%
   assert pixel_diff >= 0.10
   ```
   用计算机视觉验证虚拟机的交互响应——这是整个 TBench 最创新的评分方式。

4. **核心文件哈希比对**（install-windows-3.11）：
   从 QEMU 镜像中提取 7 个 Windows 3.11 核心文件（WIN.COM, PROGMAN.EXE, WINFILE.EXE 等），逐一 SHA256 比对，至少 5 个匹配。验证 QEMU 确实在运行真实的 Windows 3.11。

---

## 数据流全景

```
task.toml ──→ Dockerfile ──→ instruction.md ──→ Agent 执行 ──→ test.sh ──→ test_outputs.py ──→ reward.txt
  元数据参数     环境初始化      指令输入        命令/脚本/输出    统一模板      九种评分模式     0 或 1
  timeout/cpus   apt/pip/文件   风格多样        错误恢复尝试     uvx pytest     精确/哈希/API     二值文件
                                                                      + CTRF 报告
```

这条链路的每一步都有明确的设计意图：task.toml 定义约束，Dockerfile 保证一致性，instruction.md 测试信息补全，solve.sh 提供参照系，test.sh 标准化评分，test_outputs.py 精确验证，reward.txt 二值判定。

---

## Key Takeaways

1. TBench 的工程链路比题目本身更重要——可信评测需要可信链路
2. 双轨时间估计（expert/junior）为 Agent 表现提供了"人类参照系"，40x 比率意味着专业知识价值指数性
3. instruction.md 不统一风格——这本身就是对 Agent 信息补全能力的测试
4. 评分模板高度标准化（89 个任务几乎完全相同的 test.sh），但评分逻辑有九种模式适配不同任务类型
5. 四种评分防作弊武器确保"完成"意味着"真正完成"，而非投机取巧