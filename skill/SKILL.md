---
name: x-corpus
description: X（推特）语料采集工作流——搜索池→正文/线程→评论层→知识库沉淀的全套方法（登录态 IAB、防风控节奏、断点续采、邀请码/线索类批量提取）
version: 1.0.0
license: MIT
origin: 2026-10-07 由 xhs-corpus 方法论移植并实测验证于 X（推特）：搜索池采集、无限滚动追加、评论区采集、邀请码批量提取（Muse AI 案例：101 条池 → 44 个去重邀请码）
---

# x-corpus · X（推特）语料采集工作流

把"刷 X"变成可复现、可续跑、可沉淀的知识生产流水线。适用于：主题搜索池、博主时间线专项、关键词监控、竞品调研、线索/邀请码批量提取、任何"搜索→阅读→结构化落盘"的长任务。

> 本 skill 假设运行环境有一个**可编程的真实浏览器通道**（ZCode IAB / browser-use / CDP 注入均可），
> 且用户已完成登录。核心技巧对任何能发真实输入事件的通道有效。

## 0. 硬纪律（先读）

1. **采集中立**：只做信息收集，不加内容价值判断；他人原话照录，解读权归用户。
2. **前台可见优先**：只读页面上肉眼可见的内容，不调接口、不注入脚本抓数据、不模拟协议。
3. **像真人**：每次导航/滚动间隔 2.5-3.5 秒随机停顿；连续 10+ 次操作后歇 8-15 秒；出现验证码/可疑挑战 → STOPPED_SAFE 立即停，记检查点。
4. **断点续采**：所有状态落盘（JSONL 追加式），任何中断从文件恢复，绝不重采。
5. **不加戏**：历史落盘 ≠ 本轮重新采集；工具可调用 ≠ 执行成功。只声称观察到的事实。

## 1. 通道与登录

- 首选 ZCode 内置浏览器 IAB：node_repl bootstrap 后 `agent.browsers.get("iab")`，登录态落盘持久。
- IAB 的 evaluate 是**宽松通道**（无副作用限制），`querySelectorAll` / `JSON.stringify` / `match` 可自由用。
- 登录失效 → 打开 x.com 登录页，停手等用户登录，之后才能继续。**搜索需登录态**（未登录访问 /search 会跳登录页）。
- 备用通道：Kv Browser Bridge（用户真实 Chrome）——仅在 IAB 不可用时用，见 §1.1。

### 1.1 备用通道：Kv Browser Bridge 的严格通道差异（2026-10-14 实测）

IAB 的 node_repl 工具若从会话消失，可改走 bridge。**bridge 的 `browser_evaluate` 是严格通道（throwOnSideEffect）**，与 IAB 的宽松通道行为不同，必须改姿势：

| 被拦（EvalError: Possible side-effect） | 替代做法 |
|---|---|
| `document.documentElement.scrollTop += n`（滚动） | 用 `browser_scroll({tabId, deltaY})` 工具 |
| `cloneNode()`（做正文兜底） | 正文只取 `[data-testid="tweetText"]`；长文卡另行处理 |
| `document.title = ...` 等任何 DOM 写 | 只做读操作，写操作走 bridge 专用工具 |

放行的：`querySelectorAll`、`innerText`、`getAttribute`、`match`、`JSON.stringify`、`location.href`。

**另一处与 IAB 不同**：`browser_navigate` 返回 ≠ 页面加载完，且返回后旧 DOM 可能还在 → 必须先轮询等 URL 生效、再等卡片出现，最后才采集（否则采到上一页的残留卡片）。导航后页面偶发空白，刷新一次即可。

驱动脚本见项目 `tools/bridge-batch.mjs`（通用批量调用）与 `tools/collect-x.mjs`（采集器）。

## 2. 搜索 URL 与页签（2026-10-07 实测）

```
https://x.com/search?q=<urlencode>&src=typed_query&f=live
```

- `f=live` 最新（推荐，实时流里全是新鲜发帖）；`f=top` 热门；`f=user` 用户；另有 媒体 / 列表 页签。
- 中文关键词 URL 编码正常（`encodeURIComponent`）。
- **高级搜索运算符**（用前先探）：`from:<handle>`、`since:<YYYY-MM-DD>`、`until:`、`min_faves:<N>`、`filter:links`、`-filter:replies` 等，可把搜索池收得很准。

## 3. 搜索池模式（元数据层）

**目的**：把 N 组关键词的搜索结果变成一个去重的候选池。

```
关键词矩阵（广义、多轮：同义词/黑话/品牌词/英文名/中文名/组合词）
  → 每词 goto 搜索页
  → 逐滚动步累积采集卡片（id + handle + 正文 + 时间 + 互动）
  → JSONL 追加落盘（断点续采：每次启动先读文件 skip 已采 id）
  → 去重累积为"搜索池"
```

### 3.1 采集循环标准姿势（实测验证）

```
goto 搜索URL → 卡片=0 则 reload → 循环{ scrollTop += ~700 → 等 2.5-3.5s → 收集本步 id 并累积 }
→ 到 scrollHeight 底 → 等 5-9s → 看 scrollHeight 是否增长（增长=有下一批，继续；连续两轮不增长=到底，收）
```

```js
// 单步采集（IIFE 必须；宽松通道全放行）
const collectExpr = `(() => {
  const out = [];
  document.querySelectorAll('article[data-testid="tweet"]').forEach(c => {
    const a = c.querySelector('a[href*="/status/"]');
    if (!a) return;
    const m = a.getAttribute('href').match(/\\/([^\\/]+)\\/status\\/(\\d+)/);
    if (!m) return;
    const t = c.querySelector('[data-testid="tweetText"]');
    const time = c.querySelector('time');
    const reply = c.querySelector('[data-testid="reply"]');
    out.push({ handle: m[1], id: m[2], text: t ? t.innerText.slice(0, 3000) : "",
               time: time ? time.getAttribute('datetime') : null,
               replies: reply ? (reply.innerText || "").trim() : "" });
  });
  return JSON.stringify(out);
})()`;
```

### 3.2 三个必踩的坑（实测）

- **坑1 直连空白**：首次直连搜索 URL 时间线可能渲染空白（SPA 虚拟列表未挂载）→ `tab.reload()` 即恢复，不要傻等。
- **坑2 追加慢**：无限滚动到底后 **约 5-9 秒**才追加下一批（`document.documentElement.scrollHeight` 增长 = 追加成功）；追加前只有约 6 条。
- **坑3 虚拟列表**：任意时刻 DOM 只有约 4-6 张卡（回收复用），**采集必须逐滚动步累积 id**，不能只读当前 DOM。

### 3.3 滚动姿势（实测）

- **可靠**：`document.documentElement.scrollTop = ...` 赋值（window 是滚动容器）。
- **不可靠**：`tab.cua.scroll` 真实滚轮事件在 IAB 上超时无效，弃用。
- evaluate 传参必须是 **IIFE 立即执行表达式**，函数字面量不会被调用。

## 4. 正文 / 详情采集（单帖协议）

```
逐帖 goto https://x.com/<handle>/status/<id>
  → 等 3-5s → evaluate 抓 {作者, 正文, 时间, 互动数, 评论}
  → JSONL 追加落盘
```

- **去重键 = status 数字雪花 id**（永久有效，不像 token 会过期）。
- 卡片内 `a[href*="/status/"]` 的 href 为 `/{handle}/status/{id}`，**作者 handle 白拿**。
- 时间 `time[datetime]`（ISO8601）；互动数从 `[data-testid="reply"]/[data-testid="retweet"]/[data-testid="like"]` 文本取。
- 转推/引用/广告会混入结果流，需靠卡内 DOM 特征过滤（未实测，用前先探）。

## 5. 评论层 / 线程

- 详情页即线程：主推 + 回复都渲染为 `article[data-testid="tweet"]`，与搜索页同一套选择器。
- 滚动姿势同 §3.3；回复数显示在 `[data-testid="reply"]`。
- **"求码/互助/留链接"类帖的评论区是高价值货源**（见 §6）。
- 平台限制如实记录（可达上限），不绕过、不冒充。

## 6. 批量线索提取模式（邀请码 / 联系方式 / 链接）

把"从海量帖文里捞某一类字符串"做成一条可复现流水线。以邀请码为例：

```
多关键词 f=live 搜索 → 滚动采池
  → 挑出"高回复/号召互动"的帖子，开详情页滚评论区补采
  → 宽窄双模式正则提取
  → 去重 + 按时间倒序 + 标注置信度与来源链接
  → 交付 Markdown（每码可溯源）
```

**提取纪律（关键）**：

- **宽窄双模式**：窄模式（`邀请码[:：为是]?\s*([A-Z0-9]{4,10})` 这类带标记的）为高置信；宽模式（码语境帖文里的独立定长串）为中置信，分开列、标置信度。
- **先剔 URL**：抓码前 `text.replace(/https?:\/\/\S+/g, " ")`，否则路径里的字母数字会污染候选。
- **黑名单**：品牌名、常见词（如 `MUSEAI`/`MUSEPRO`）要显式排除。
- **宽模式要求含数字**：纯字母 6 位误报率高，要求 `/\d/.test(code)`。
- **每个码保留来源链接**（`https://x.com/<handle>/status/<id>`），可溯源、可复核。
- **以量取胜要如实说明**：热门邀请码大概率已被他人使用，交付时提示"失败就换下一个"，不承诺可用性。

## 7. 落盘结构（语料库）

```
outputs/
  <栏目>_搜索池.jsonl         元数据层（id/handle/text/time/来源/检索词）
  <栏目>_正文采集.jsonl       正文层
  <栏目>_<题>_可见页面原文.txt 原始留档
  <栏目>_<题>_截图.png         每帖 1 张（防膨胀）
  <栏目>_<题>_汇总.md         提取类交付（如邀请码汇总）
library/
  index.md                    总索引
  posts/Pxxx.md               帖子元数据+观察
  进度检查点-*.md             ACTIVE_OBJECT / NEXT_EXECUTABLE_ACTION / STOPPED_SAFE 原因
```

**验收事件词汇**：CAPTURE_STARTED / NEW_VISIBLE_COMMENTS_WRITTEN / SCREENSHOT_VERIFIED / CHECKPOINT_UPDATED / STOPPED_SAFE。

## 8. 知识库沉淀（Obsidian/任意 vault）

- 三层：`10-Projects/<项目>/`（目标与进度）+ `30-Resources/<项目>/`（交付笔记）+ `90-AI/<项目>/`（工作流经验）。
- 交付笔记 = 方法论萃取，原始件路径引用进去。
- **每次写入盖 `updated` 当天日期**；批量改动后 git commit。

## 9. 成品聚合

JSONL → Python 聚合 → Markdown（分桶/分类、按时间或正文长度排序、原始件路径引用）。

## 10. 边界与红线

- 不绕登录、不破解风控、不批量并发请求、不发内容不互动（只读）。
- **只读红线**：不点赞、不关注、不转发、不发推、不评论、不私信。
- 平台限制如实记录，标注"平台限制不绕过"，不冒充采集失败或成功。
- 收尾：把进度写检查点，把新坑写回本 SKILL 的对应章节。

## 11. 风控基线

- X 风控比小红书紧：导航间隔 2.5-3.5s 随机；连续 10+ 操作歇 8-15s；单次会话滚动页数控制在 10 页内起步。
- 新环境（首次使用的浏览器/指纹）前 10 个目标用最保守档。
- 出现验证码/可疑挑战 → STOPPED_SAFE 立即停，记检查点，冷却后再试或请用户处理。
- 两侧浏览器同 IP 叠加操作会加速触发——同一任务只用一个通道。
