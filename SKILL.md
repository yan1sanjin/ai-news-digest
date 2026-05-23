---
name: ai-news-digest
description: 抓取主流中外 AI 资讯源生成中文 markdown 日报。用户说"生成今日 AI 日报"/"今天有什么 AI 新闻"等指令时启用。
allowed-tools: WebFetch, WebSearch, Read, Write, Bash(mkdir *), Bash(date *), Bash(ls *)
---

# AI News Digest Skill

You are generating today's AI industry news digest in Chinese-friendly format.

---

## ⚠️ 核心原则(v3.1 新增 · 高于一切其他规则)

**信源真实性 > 信号广度。宁可漏报,不要错报。**

WebSearch 的二手报道可能有 3 类污染:
1. 媒体夸张标题(如"全面换装""彻底颠覆"等营销语)
2. SEO 投毒 / 虚构页面(看似官方域名但实际是仿冒/虚构内容)
3. AI 生成的虚假新闻被搜索引擎索引

**任何 T+1 信号在写入最终报告前必须通过 Step 6.5 事实核查**。无法 verify 的内容 → 标⚠️或剔除,不要为了凑条数硬上。

---

## Step 1: Determine output path

Default path: `~/Desktop/ai-news/YYYY-MM-DD.md`,使用今天的日期(YYYY-MM-DD 格式)。

如果用户在调用时提供了 `path=` 参数,使用用户指定路径。

如果该路径已存在文件,改用 `YYYY-MM-DD-second.md`(再有则 `-third.md`),**永不覆盖**。

**确保父目录存在** (无论是默认路径还是 `path=` 自定义路径, 父目录不存在都先创建)。先判断当前 OS 用对应命令:

**macOS / Linux / WSL / Git Bash** (bash):
```bash
mkdir -p "$(dirname <最终目标路径>)"
```

**Windows PowerShell**:
```powershell
New-Item -ItemType Directory -Force -Path (Split-Path -Path "<最终目标路径>" -Parent) | Out-Null
```

具体例子:
- 默认 Unix: `mkdir -p ~/Desktop/ai-news/`
- 默认 Windows: `New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\Desktop\ai-news"`
- 自定义 Unix `path=~/Documents/digests/today.md` → `mkdir -p ~/Documents/digests/`
- 自定义 Windows `path=$env:USERPROFILE\Documents\digests\today.md` → `New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\Documents\digests"`

**Windows 默认输出路径用 `$env:USERPROFILE\Desktop\ai-news\YYYY-MM-DD.md`** (PowerShell 也支持 `~` 但 `$env:USERPROFILE` 更明确稳定)。

获取今天日期 (`YYYY-MM-DD` 格式):

**Unix**:
```bash
date +%Y-%m-%d
```

**Windows PowerShell**:
```powershell
Get-Date -Format "yyyy-MM-dd"
```

**配置说明**:
- 默认输出目录可在 SKILL.md 顶部自行替换 (例如改为 `~/Documents/ai-news/`),改完后下面 Step 1.5 / Step 6.7 的路径要同步改
- 运行时通过 `path=` 参数可临时覆盖默认值
- 日期基于运行机器的本地时区, 跨时区团队请注意

---

## Step 1.5: Read recent 3-day digests for dedup baseline

如果 `~/Desktop/ai-news/` (或自定义默认目录) 下存在过去 3 天的日报 (`YYYY-MM-DD.md` 命名), 用 Read 工具读取这些文件, 提取已出现的新闻**标题列表**作为去重 baseline (不需要读全文, 标题列表即可)。

Step 6.1 去重时, 把当天候选条目跟 baseline 对比, 标题语义相似度 ≥ 70% 的视为重复, 剔除。

过去 3 天若没有日报文件, 跳过本步, 不影响后续流程。

---

## Step 2: Fetch Tier 1 (English, must · all 5)

WebFetch each source。**不同源用不同 prompt, 不要套统一模板**:

1. **Hacker News Front Page** · https://news.ycombinator.com/
   - Prompt: `提取首页 top 30 stories 里 AI 相关的 (Claude/GPT/Gemini/LLM/Agent/RAG/MCP/embedding/foundation model/AGI/Anthropic/OpenAI/DeepSeek/Qwen/HuggingFace/diffusion/tokenizer/fine-tuning/reasoning model 等关键词), 每条返回 标题/URL/分数/comments 数/1 句要点`

2. **Anthropic News** · https://www.anthropic.com/news
   - Prompt: `提取最新 5 篇 announcement, 每条返回 标题/URL/发布日期/1 句核心要点`

3. **OpenAI 动态** · ⚠️ openai.com/news/ 直接 WebFetch 会 403 → **改用 WebSearch** 关键词:`OpenAI announcement 2026 latest` 或 `"OpenAI" news this week`(在 Step 5 一并跑)

4. **Latent Space** · https://www.latent.space/feed (RSS, 主页 WebFetch 抓不到列表)
   - Prompt: `提取 RSS feed 最新 3 篇文章, 每条返回 标题/URL/发布日期/1 句要点`

5. **HuggingFace Blog** · https://huggingface.co/blog
   - Prompt: `提取最新 5 篇 blog post, 每条返回 标题/URL/发布日期/1 句要点`

---

## Step 3: Fetch Tier 2 (Chinese, must · all 3)

WebFetch, 每个中文源单独写 prompt (不要套统一模板):

- **量子位** · https://www.qbitai.com/
  - Prompt: `提取首页今天最新的 5-10 条 AI 资讯文章, 每条返回 标题/URL/发布时间/1 句核心要点`

- **机器之心** · https://www.jiqizhixin.com/
  - Prompt: `提取首页今天最新的 5-10 条 AI 资讯, 每条返回 标题/URL/发布时间/1 句核心要点`

- **36 氪 AI 频道** · https://36kr.com/information/AI/
  - Prompt: `提取首页今天最新的 5-10 条 AI 类资讯, 每条返回 标题/URL/发布时间/1 句核心要点`

---

## Step 4: Fetch Tier 3 (rotation pool · pick 2 of 4)

根据 Tier 1/2 抓到的内容判断今天的热点方向,从下面 4 个里**挑 2 个最相关的**抓 (轮换池, 用通用模板即可):

| 源 | URL | 何时该选 |
|---|---|---|
| Google AI Blog | https://blog.google/technology/ai/ | Tier 1 出现 Google/DeepMind 相关消息时 |
| TechCrunch AI | https://techcrunch.com/category/artificial-intelligence/ | 出现融资 / 商业 / 产品发布消息时 |
| The Verge AI | https://www.theverge.com/ai-artificial-intelligence | 出现主流媒体级别热点时 |
| One Useful Thing | https://www.oneusefulthing.org/ | 出现应用层 / 思想 / 评估方法时 |

通用 WebFetch prompt: `提取页面上最新的 5-10 条 AI 相关内容, 每条返回 标题/URL/发布时间/1 句核心要点`

**保底规则**:如果完全没有明显热点方向,默认选 `Google AI Blog` + `TechCrunch AI`(覆盖商业模型 + 融资动态)。

⚠️ **注意**:Reddit 域名(www.reddit.com)被 Claude Code WebFetch 硬屏蔽,**不能直接抓**。当天若主题涉及开源/本地模型,在 Step 5 加 `r/LocalLLaMA top posts` 类关键词替代。

---

## Step 5: Tier 4 · T+1 主体配套搜索 (WebSearch)

基于 T+0 直抓 / T+1 WebSearch 双引擎模式,主体源每个跑一条配套 WebSearch 抓**官方主页漏掉的 3 类内容**:
1. 子站 / 子页面发布(如 red.anthropic.com 的模型预览)
2. 第三方独家报道(TechCrunch / VentureBeat 类)
3. 被投资 / 被收购等"自己不发"的新闻

### 5.1 主体配套搜索(必跑 5-6 个)

按 Tier 1/2 主体跑配套 WebSearch:

| 主体 | 配套 WebSearch 关键词 |
|---|---|
| Anthropic | `Anthropic news this week 2026` 或 `Anthropic announcement latest` |
| OpenAI(替代直抓 403) | `OpenAI announcement 2026 latest` |
| HuggingFace | `HuggingFace top releases this week` |
| Latent Space(补 RSS 不全) | `Latent Space podcast latest 2026` |
| 国产开源动态 | `DeepSeek OR Qwen OR 小米 大模型 最新` |
| Reddit 替代 | `r/LocalLLaMA top posts new model release` |

### 5.2 突发热点搜索(动态 2-3 个)

基于 Tier 1-3 抓到的信号,跑 2-3 个突发关键词:
- 当天反复出现的公司/模型 → 跑该名字
- 例:`<某新模型> release` / `<某收购事件> details`

**保底关键词**(完全没明显热点时):
- `AI news today`
- `Claude OR GPT OR Gemini latest`

---

## Step 6: Filter, dedupe, and choose authoritative link (T+0 优先)

整合 Tier 1-4 抓到的所有内容,做去重 + 链接择优 + 排序 + 筛选:

### 6.1 去重 + 链接择优(关键新规则)

同一新闻被多个引擎抓到 → **优先用 T+0 直抓的官方原文链接**,T+1 WebSearch 仅作交叉验证或备份。

**链接优先级**:
1. 🥇 T+0 直抓的官方源(anthropic.com / openai.com / huggingface.co 等)
2. 🥈 第一手报道(TechCrunch / Bloomberg / The Verge 等知名媒体的独家)
3. 🥉 二手解读 / 列表(VentureBeat / Fool / Releasebot 等聚合站)

**去重逻辑**:
- 标题/主题相似 → 合并为一条
- 保留信息最完整的描述 + 用最高优先级的链接
- **已经在 Step 1.5 baseline 里的 (语义相似度 ≥ 70%)** → 剔除, 不放入今天日报
- 在底部"参考来源"小字标注其他报道源(可选)

**过滤掉**:
- 纯 tutorial / 教程类
- 求 GitHub star 的项目
- 学术论文(没有产品落地暗示的纯研究)
- 纯 listicle(top 10 / best of)

**排序优先级**(从高到低):
1. 行业级影响(产品发布 / 公司战略 / 监管政策)
2. 技术新颖性(新架构 / 新方法 / 新模型)
3. 商业信号(融资 / 收购 / 合作)
4. 通用 curiosity 内容

**最终选 10-15 条**,分到 3 个 section:
- 一、今日要闻 (5-8 条 · 信号最强)
- 二、技术热点 (3-5 条 · 模型/工具/基础设施)
- 三、行业动态 (3-5 条 · 商业/融资/公司动态)

---

## Step 6.5 · 信源可信度评估 + 事实核查(v3.1 新增 · 强制执行)

**对 Step 6 筛选后的每条候选条目,先做信源评级,再决定是否保留**。

### 6.5.1 三档评级

| 等级 | 定义 | 处理 |
|---|---|---|
| 🟢 **高可信** | T+0 直抓官方源,且原文有清楚表述支持你写的内容 | 直接保留 |
| 🟡 **中可信** | T+1 多家媒体交叉报道,内容一致 + 来自白名单域名 | 必须 fetch 一次最权威的来源 verify · 通过则保留 |
| 🔴 **低可信** | 单一 T+1 来源 / 营销夸张标题 / 来自黑名单域名 / 数字异常大 / Anthropic 官方 News 没列但二手有 | **默认剔除**,fetch verify 后能 verify 的升级为 🟡 |

### 6.5.2 域名白名单(可作 verify 源)

**官方第一手**(.com/.cn 主域):
- anthropic.com / openai.com / huggingface.co / deepseek.com / qwen.alibaba.com 等模型公司主域
- google.com / blog.google / aws.amazon.com 等云厂商
- github.blog(GitHub 官方)

**知名媒体**(可作交叉验证):
- 国际:Bloomberg / Reuters / TechCrunch / The Verge / WIRED / VentureBeat / CNBC / Axios / The Information
- 国内:36 氪 / 量子位 / 机器之心 / 钛媒体 / 品玩 / 新浪财经 / 界面新闻

### 6.5.3 域名黑名单(纯 SEO 站,不可信)

**绝对不要使用,即使 WebSearch 返回**:
- fazm.ai / aitooldiscovery.com / releasebot.io / felloai.com
- letsdatascience.com / ghost.codersera.com / igmguru.com
- techaimag.com / theneuron.ai 这类聚合 SEO 站
- 任何 .ai / .blog 后缀但作者匿名 / 无明确组织背景的站
- 任何子域名异常的"看似官方"页面(如 red.anthropic.com 这种 v3 实测被识别为虚构的)

### 6.5.4 数字与名称的合理性核查

**红旗信号**(出现就要重新 verify):
- 数字异常大($100B+ 募资 / 万亿参数等) → 必须 fetch 官方公告确认
- 公司"全面换装"、"彻底颠覆"、"告别 X"等营销语 → 大概率是夸张标题
- 没听说过的产品名 + 没在官方 News 出现 → 可能是 SEO 投毒
- 同一新闻只有 1 个二手来源,没有任何官方/Tier 1 媒体跟进 → 可疑

### 6.5.5 实战工作流

```
对每条候选条目:
  1. 评级 → 🟢 / 🟡 / 🔴
  2. 🔴 → 直接剔除
  3. 🟡 → fetch 一次最权威源(优先官方,其次白名单媒体)
     - fetch 内容支持 → 升级为🟢保留 + 用最权威链接
     - fetch 不支持 / 链接 404 / 403 → 降为 🔴 剔除 OR 标 ⚠️ "未 verify"
  4. 🟢 → 直接保留

最终原则:
- 宁可日报少 5 条都是 🟢/🟡 verified,
- 也不要 19 条里有 6 条 🔴 未 verify(这就是 v3 翻车的具体场景)
```

---

## Step 6.7 · Fail-safe threshold check

**如果 Tier 1 (英文官方源) 成功抓取的源 < 2 个, 整个任务 abort**, 写入最简错误报告并退出, 不要继续到 Step 7-9。

错误报告格式:

```markdown
# AI 资讯日报 · YYYY-MM-DD (生成失败)

> Tier 1 抓取失败 (成功源 < 2),
> 为避免输出全中文的"全球 AI 日报"误导用户, 任务终止。

**失败源**:
- [源名] · 原因:[404 / timeout / robots / 其他]
- ...

**成功源**(本来应该用上但因 Tier 1 不足放弃):
- [Tier 2 / Tier 3 源列表]

**建议**: 检查网络 / 各源 URL 是否变化 / Anthropic WebFetch 是否正常, 稍后重试。
```

写入路径同 Step 1 (`~/Desktop/ai-news/YYYY-MM-DD.md` 或 `path=` 指定)。

**设计原则**: Tier 1 是这个 skill 的事实锚, Tier 1 失守 → 输出全中文的"全球 AI 日报"会误导用户, 宁可任务失败也不出劣质日报。

---

## Step 7: Translate English entries

**英文源条目** 必须做以下处理:
- ✅ 保留原英文标题作为 markdown 链接:`[Original English Title](URL)`
- ✅ **下一行加** `**[中文译]**:` + 中文翻译标题
- ✅ 摘要(`**摘要**:`)用**中文**写,无论原文什么语言
- ✅ "为什么值得关注"(`**为什么值得关注**:`)用**中文**写

**中文源条目**:
- 标题保持中文(不需要翻译,不需要 `[中文译]` 行)
- 摘要保持中文

---

## Step 8: Generate trend summary (3-5 themes)

读完所有筛选后的条目,识别 3-5 个底层主题。每个主题:
- 一句加粗的中文主题句
- 一句中文解释这个跨条目的模式
- (可选)引用 1-2 条具体新闻作证据

**好主题的特征**:
- 跨多条新闻的共同模式(不是单条新闻总结)
- 揭示行业动向(不是表面描述)
- 具体可验证(不是空泛感慨)

**好主题示例**:
- "**长上下文进入工业级**:Anthropic 1M + 国产玩家也在跟进,长文档类应用门槛大降"
- "**Agent 评估工具开始独立成赛道**:LangFuse / Braintrust 都拿到融资"
- "**国产模型在数学推理上进入第一梯队**:DeepSeek R1 + 腾讯混元 + 通义都在 GSM8K 接近 95%"

**避免的坏主题**:
- "AI 发展很快"(空泛)
- "今天有很多 OpenAI 新闻"(描述不是判断)
- "未来值得期待"(无信息量)

---

## Step 9: Write the markdown file

按以下完整格式写入文件:

```markdown
# AI 资讯日报 · YYYY-MM-DD

> 生成时间:YYYY-MM-DD HH:MM
> 资讯源:HN / Anthropic / OpenAI / Latent Space / HuggingFace / 量子位 / 机器之心 / 36 氪 + 轮换 [X] / [Y]
> 共 N 条精选

---

## 一、今日要闻 (5-8 条)

### [Original English Title](URL)
**[中文译]**:中文翻译标题(英文源必填)
**来源**:Anthropic News
**摘要**:1-2 句中文摘要
**为什么值得关注**:1 句中文(可选)

### [中文原标题](URL)
**来源**:量子位
**摘要**:1-2 句中文摘要(中文源不需要 [中文译] 行)
**为什么值得关注**:1 句中文(可选)

---

## 二、技术热点 (3-5 条)

[同上格式]

---

## 三、行业动态 (3-5 条)

[同上格式]

---

## 四、今日趋势总结

- **主题 1 · [简短主题词]**:一句话中文归纳跨条目的模式
- **主题 2 · [简短主题词]**:同上
- **主题 3 · [简短主题词]**:同上

---

**今日值得深读 (可选)**:
- [文章标题](URL) —— 一句话中文推荐理由

---

**抓取失败记录**(如有):
- [源名] · 原因:[404 / timeout / robots]

---

*Generated by ai-news-digest skill · Tier 1+2 必查 · Tier 3 选用了 [X] / [Y]*
```

---

## Step 10: Confirm to user

写完文件后,打印一句中文确认:

```
AI 资讯日报已生成: <完整路径>
共 N 条资讯 · M 个趋势主题
英文条目 X 条(已翻译) · 中文条目 Y 条
Tier 3 选用了:<两个源名>
```

---

## Constraints (汇总)

- 不要把全部抓到的内容塞进文件 —— 只保留 10-15 条精选
- 中文源保持原文 / 英文源必加 `[中文译]` 标题 + 中文摘要(用户主要诉求)
- 不要重复总结同一新闻
- 摘要必须 ≤ 2 句话
- 趋势总结必须基于今天抓到的真实内容,**不要编造行业判断**
- 单个 Tier 1/2 源抓取失败(404 / 超时 / robots 拦截)→ 在文件末尾"抓取失败"小节记录, 继续其他源
- **但** Tier 1 < 2 个源成功 → 触发 Step 6.7 abort, 不要继续 Step 7-9
- URL 必须是真实抓到的,不要凭记忆构造
- 标题必须忠于原文(英文翻译要准,不要发挥)

---

## 调用示例

用户调用方式(任一):
```
生成今日 AI 日报
```
```
用 ai-news-digest 跑一下今天的资讯
```
```
今天有什么 AI 新闻
```
```
ai-news-digest path=/path/to/custom.md
```

Claude 基于 description 自动识别意图并调用本 skill。
