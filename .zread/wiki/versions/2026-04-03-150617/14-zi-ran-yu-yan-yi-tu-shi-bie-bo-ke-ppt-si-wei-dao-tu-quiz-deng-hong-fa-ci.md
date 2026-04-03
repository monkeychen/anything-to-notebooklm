本页深入解析 anything-to-notebooklm Skill 如何将用户的自然语言表述映射为 NotebookLM 的具体生成命令。当你对 Claude Code 说"把这篇文章做成播客"或"帮我画个思维导图"时，系统需要精确识别你的输出格式意图，并将其转换为对应的 `notebooklm generate` 子命令。本页将完整呈现 9 种意图的触发词词典、映射机制以及意图未命中时的默认行为。

Sources: [SKILL.md](SKILL.md#L121-L135)

## 意图识别的定位：工作流中的"翻译层"

在整个处理流水线中，意图识别位于**内容获取与上传之后、生成与下载之前**，是一个关键的"翻译层"。它不负责判断内容源类型（那是 Step 1 的工作），而是专注于从用户的自然语言中提取**"要生成什么格式的输出"**这一维度。

下面的流程图展示了意图识别在整个工作流中的精确位置：

```mermaid
flowchart LR
    A["用户自然语言输入"] --> B["Step 1: 内容源识别"]
    B --> C["Step 2: 内容获取"]
    C --> D["Step 3: 上传到 NotebookLM"]
    D --> E["🎯 Step 5: 意图识别<br/>（本页主题）"]
    E -->|识别到意图| F["调用 generate 命令"]
    E -->|未识别意图| G["仅上传，等待后续指令"]
    F --> H["artifact wait 等待完成"]
    H --> I["download 下载产物"]
```

需要注意的是，SKILL.md 中将意图识别标注为 Step 5（标记为"可选"），这意味着**系统允许用户只上传内容而不立即生成任何格式**——当你没有明确说"生成什么"时，系统默认行为就是如此。

Sources: [SKILL.md](SKILL.md#L218-L222), [SKILL.md](SKILL.md#L134-L135)

## 9 种输出格式的触发词完整词典

系统通过一张**自然语言 → 意图关键词 → NotebookLM 命令**的三层映射表来实现意图识别。下表列出了所有 9 种支持的输出格式及其对应的触发短语：

| 识别意图 | 用户可能说的话（触发词） | NotebookLM 命令 | 典型用途 |
|---------|----------------------|----------------|---------|
| **audio** | "生成播客" / "做成音频" / "转成语音" | `generate audio` | 通勤路上听文章 |
| **slide-deck** | "做成PPT" / "生成幻灯片" / "做个演示" | `generate slide-deck` | 团队分享、会议汇报 |
| **mind-map** | "画个思维导图" / "生成脑图" / "做个导图" | `generate mind-map` | 梳理概念结构 |
| **quiz** | "生成Quiz" / "出题" / "做个测验" | `generate quiz` | 自测知识掌握程度 |
| **video** | "做个视频" / "生成视频" | `generate video` | 可视化内容呈现 |
| **report** | "生成报告" / "写个总结" / "整理成文档" | `generate report` | 深度分析、综合研究 |
| **infographic** | "做个信息图" / "可视化" | `generate infographic` | 数据可视化呈现 |
| **data-table** | "生成数据表" / "做个表格" | `generate data-table` | 结构化数据整理 |
| **flashcards** | "做成闪卡" / "生成记忆卡片" | `generate flashcards` | 记忆巩固、考前复习 |

Sources: [SKILL.md](SKILL.md#L121-L134)

### 触发词的设计逻辑

观察上表可以发现，触发词的设计遵循几个清晰的**语义模式**：

- **动作动词 + 格式名词**：如"生成播客"、"做成PPT"、"画个思维导图"——这是最常见的模式，动作词（生成/做成/画个/做个/整理成）与格式词直接组合。
- **同义词覆盖**：每种意图都有 2-3 个同义触发词，确保用户用不同的说法都能被识别。例如"思维导图"、"脑图"、"导图"三个词指向同一个意图。
- **格式转换语义**：如"转成语音"、"整理成文档"这类表达暗示了格式转换，也被纳入对应的意图。

Sources: [SKILL.md](SKILL.md#L121-L134)

## 意图到生成命令的执行链路

当系统成功识别到某个意图后，会触发一个标准的三步执行链路。下表展示了每种意图对应的完整执行细节，包括等待命令和下载产物的文件格式：

| 意图 | 发起命令 | 等待完成 | 下载命令 | 产物格式 | 典型耗时 |
|------|---------|---------|---------|---------|---------|
| audio | `notebooklm generate audio` | `artifact wait` | `download audio ./output.mp3` | `.mp3` | 2-5 分钟 |
| slide-deck | `notebooklm generate slide-deck` | `artifact wait` | `download slide-deck ./output.pdf` | `.pdf` | 1-3 分钟 |
| mind-map | `notebooklm generate mind-map` | `artifact wait` | `download mind-map ./map.json` | `.json` | 1-2 分钟 |
| quiz | `notebooklm generate quiz` | `artifact wait` | `download quiz ./quiz.md --format markdown` | `.md` | 1-2 分钟 |
| video | `notebooklm generate video` | `artifact wait` | `download video ./output.mp4` | `.mp4` | 3-8 分钟 |
| report | `notebooklm generate report` | `artifact wait` | `download report ./report.md` | `.md` | 2-4 分钟 |
| infographic | `notebooklm generate infographic` | `artifact wait` | `download infographic ./infographic.png` | `.png` | 2-3 分钟 |
| flashcards | `notebooklm generate flashcards` | `artifact wait` | `download flashcards ./cards.md --format markdown` | `.md` | 1-2 分钟 |

Sources: [SKILL.md](SKILL.md#L222-L232), [SKILL.md](SKILL.md#L512-L517)

关于执行链路，有三个关键细节值得注意：

**第一，等待是强制性的。** 生成请求发起后会返回一个 `task_id`，必须调用 `artifact wait <task_id>` 等待处理完成，才能执行下载。如果跳过等待直接下载，会导致失败。

**第二，不同意图的产物格式不同。** 播客产出 `.mp3`，PPT 产出 `.pdf`，思维导图产出 `.json`，Quiz 和闪卡产出 `.md`——这些格式由 NotebookLM API 决定，不可自定义。

**第三，视频生成耗时最长。** 从上表的典型耗时可以看出，video 格式需要 3-8 分钟，而 mind-map 和 quiz 通常只需 1-2 分钟。在实际使用中需要合理预期等待时间。

Sources: [SKILL.md](SKILL.md#L233-L238)

## 意图未命中时的默认行为

系统设计了一个明确的兜底策略：**如果用户没有在输入中包含任何可识别的触发词，默认只上传内容到 NotebookLM，不执行任何生成操作**。

这意味着以下这些输入**不会触发任何生成**：

```
"把这篇文章传到NotebookLM"          → 仅上传
"帮我分析这个网页 https://..."       → 仅上传
"把这个PDF上传到NotebookLM"          → 仅上传
```

而以下这些输入**会触发对应格式的生成**：

```
"把这篇文章生成播客"                  → 上传 + generate audio
"这个PDF帮我做成PPT"                  → 上传 + generate slide-deck
"帮我分析这个网页，整理成文档"         → 上传 + generate report
```

关键区别在于：用户的表述中是否包含"播客"、"PPT"、"报告"等格式触发词。系统会在上传完成后等待用户的后续指令，你可以随时追加生成请求。

Sources: [SKILL.md](SKILL.md#L134-L135)

## 触发词使用的实战场景

为了帮助理解触发词在实际对话中的使用方式，以下展示 4 个典型场景，涵盖不同内容源与不同意图的组合：

### 场景 1：微信文章 → 播客（audio 意图）

```
用户输入：把这篇文章生成播客 https://mp.weixin.qq.com/s/abc123
意图识别：audio
执行链路：MCP 抓取 → 创建 TXT → 上传 → generate audio → 等待 → 下载 .mp3
```

这是最常见的使用场景。微信文章通常 1000-5000 字，生成的播客时长约 3-8 分钟，非常适合通勤时收听。

Sources: [SKILL.md](SKILL.md#L241-L268)

### 场景 2：YouTube 视频 → 思维导图（mind-map 意图）

```
用户输入：这个视频帮我画个思维导图 https://www.youtube.com/watch?v=abc123
意图识别：mind-map
执行链路：URL 直接传递 → NotebookLM 提取字幕 → generate mind-map → 等待 → 下载 .json
```

YouTube 链接不需要经过 markitdown 转换，NotebookLM 会自动提取视频字幕和元数据。

Sources: [SKILL.md](SKILL.md#L270-L293)

### 场景 3：Markdown 文件 → Quiz（quiz 意图）

```
用户输入：这个Markdown生成Quiz /Users/joe/notes/machine_learning.md
意图识别：quiz
执行链路：直接上传 .md → generate quiz → 等待 → 下载 .md（含 10 道选择题 + 5 道简答题）
```

Markdown 文件是唯一支持直接上传的文档格式，无需经过 markitdown 转换。

Sources: [SKILL.md](SKILL.md#L381-L403)

### 场景 4：搜索关键词 → 报告（report 意图）

```
用户输入：搜索 'AI发展趋势 2026' 并生成报告
意图识别：report
执行链路：WebSearch 搜索 → 汇总前 5 条结果 → 创建 TXT → 上传 → generate report → 等待 → 下载 .md
```

搜索关键词场景中，系统会汇总 3-5 条搜索结果后整合为单个文件上传。

Sources: [SKILL.md](SKILL.md#L295-L321), [SKILL.md](SKILL.md#L193-L196)

## 触发词速查表

将所有触发词按使用频率排列，方便日常快速查阅：

| 高频使用 | 触发词 | 意图 |
|---------|-------|------|
| ⭐⭐⭐ | "生成播客" / "做成音频" | audio |
| ⭐⭐⭐ | "做成PPT" / "生成幻灯片" | slide-deck |
| ⭐⭐ | "画个思维导图" / "生成脑图" | mind-map |
| ⭐⭐ | "生成报告" / "写个总结" | report |
| ⭐ | "生成Quiz" / "出题" / "做个测验" | quiz |
| ⭐ | "做个视频" / "生成视频" | video |
| ⭐ | "做个信息图" / "可视化" | infographic |
| ⭐ | "做成闪卡" / "生成记忆卡片" | flashcards |
| ⭐ | "生成数据表" / "做个表格" | data-table |

Sources: [SKILL.md](SKILL.md#L121-L134)

## 扩展阅读

- **意图识别完成后会发生什么？** 完整的生成命令执行、等待与下载流程详见 [生成命令与产物下载：artifact wait 与 download 工作流](15-sheng-cheng-ming-ling-yu-chan-wu-xia-zai-artifact-wait-yu-download-gong-zuo-liu)
- **想一次性生成多种格式？** 多意图处理机制详见 [多意图处理：一次性生成多种格式](22-duo-yi-tu-chu-li-ci-xing-sheng-cheng-duo-chong-ge-shi)
- **想对生成结果添加自定义要求？** 自定义生成指令详见 [自定义 Notebook：指定已有笔记本或添加自定义生成指令](23-zi-ding-yi-notebook-zhi-ding-yi-you-bi-ji-ben-huo-tian-jia-zi-ding-yi-sheng-cheng-zhi-ling)
- **想了解整个处理流程的全貌？** 从自然语言到文件生成的完整数据流详见 [整体技术架构：从自然语言到文件生成的数据流](5-zheng-ti-ji-zhu-jia-gou-cong-zi-ran-yu-yan-dao-wen-jian-sheng-cheng-de-shu-ju-liu)