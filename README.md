# dailyscan

每天自动扫描 X、Reddit、雪球、Hacker News、Product Hunt、Upwork 六大平台，提炼 AI+科技趋势、产品机会、付费需求信号。

> ⚠️ 这个工具不会给你"标准答案"。它帮你把几千条信息压缩到几十条，但**你需要自己点开链接判断**。详见[这篇文章]()中的讨论。

## 它能做什么

- 打开浏览器自动采集六大平台当日内容
- 用 LLM 提炼热点、产品机会、付费需求、微机会
- 输出结构化 Markdown 报告，每条信号附带原文链接

## 它不能做什么

- **不能替你判断信息质量。** AI 检索本质上是找"语义最接近"的内容，不是"最好"的内容。报告里的链接你需要自己点开看。
- **不能保证覆盖完整。** AI 爬虫不执行 JavaScript，搜索引擎只索引约 4-10% 的互联网，大量高质量内容藏在登录墙和付费墙后面。

## 前置条件

1. **Node.js** ≥ 18
2. **Playwright CLI**：`npm install -g @playwright/cli`
3. **Edge 浏览器**（Windows 自带；Mac 需安装）
4. **Claude Code** 或支持 Skill 的 AI 编程工具
5. 六个平台的账号（见下方注册指南）
6. 科学上网（X、Reddit、HN、PH、Upwork 需要）

## 快速开始

### 第一步：注册账号

| 平台 | 注册地址 | 说明 |
|------|---------|------|
| X (Twitter) | https://x.com | 免费，关注你感兴趣的 AI/科技账号 |
| Reddit | https://reddit.com | 免费，订阅 r/artificial r/MachineLearning r/LocalLLaMA 等 |
| 雪球 | https://xueqiu.com | 免费，自选股加入 AI/半导体相关标的 |
| Hacker News | https://news.ycombinator.com | 无需注册（只读采集） |
| Product Hunt | https://producthunt.com | 免费 |
| Upwork | https://upwork.com | 免费（查看需求即可，不需要接单） |

### 第二步：首次登录（必须！）

运行以下命令，逐平台登录并保存登录状态：

```bash
# 创建持久化 profile 目录
mkdir -p ~/.claude/playwright-edge-profile

# === X (Twitter) ===
playwright-cli -s=setup-x open "https://x.com/login" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed
# 👆 浏览器会打开。手动输入账号密码登录。登录成功后看到首页时间线，关闭浏览器。
playwright-cli -s=setup-x close

# === Reddit ===
playwright-cli -s=setup-reddit open "https://old.reddit.com/login" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed
# 👆 手动登录。成功后看到首页帖子列表，关闭浏览器。
playwright-cli -s=setup-reddit close

# === 雪球 ===
playwright-cli -s=setup-xueqiu open "https://xueqiu.com/" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed
# 👆 点击右上角"登录"，用手机号/微信扫码登录。成功后关闭浏览器。
playwright-cli -s=setup-xueqiu close

# === Product Hunt ===
playwright-cli -s=setup-ph open "https://www.producthunt.com/" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed
# 👆 点击 Sign in，用 Google/GitHub 账号登录。成功后关闭浏览器。
playwright-cli -s=setup-ph close

# === Upwork ===
playwright-cli -s=setup-upwork open "https://www.upwork.com/ab/account-security/login" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed
# 👆 手动登录。成功后看到 Find Work 页面，关闭浏览器。
playwright-cli -s=setup-upwork close

echo "✅ 全部平台登录完成！Profile 保存在 ~/.claude/playwright-edge-profile/"
```

> 💡 **HN 无需登录。** 只读采集，不需要账号。
>
> 💡 **所有平台共用一个 profile**，登录态不会互相覆盖。后续日常扫描时浏览器会自动带上已保存的 cookie。
>
> 💡 **以后如果某个平台掉登录了**，重新跑上面对应的那条命令即可。

### 第三步：验证登录态

```bash
# 快速验证：打开 X 看是否已登录
playwright-cli -s=verify open "https://x.com/home" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed

# 在浏览器里确认能看到首页时间线（不是登录页），然后关闭
playwright-cli -s=verify close
```

看到首页时间线 = 登录成功。看到登录页 = cookie 过期，需要重新登录。

### 第四步：运行扫描

将 `SKILL.md` 放到你的 Claude Code 项目的 `.claude/skills/daily-scan/` 目录下，然后在对话中输入：

```
/daily-scan
```

首次运行会先检查外网可达性，然后逐平台采集，最后用 LLM 提炼并输出报告。

## 输出说明

扫描完成后，结果写入 `output/YYYY-MM-DD.md`，包含：

| 板块 | 内容 |
|------|------|
| 🔥 今日热点 | 3-5 条跨平台共振的话题 |
| 💡 产品机会信号 | 2-4 个宏观方向级别的产品缺口 |
| 💰 付费需求信号 | Upwork 上匹配你能力栈的付费项目 |
| 🔧 微机会 | 评论区中的具体可落地诉求 |
| 📊 趋势指标 | 对比前几日的升温/降温话题 |
| 🗣️ 值得深读 | 今日最有信息量的帖子 + 为什么 |

**每条信号都附带原文链接。** 这是刻意设计的——你需要点开链接，自己判断。AI 的提炼是一个压缩和筛选的过程，压缩就会有丢失，筛选就会有偏差。

## 架构说明

```
dailyscan/
├── README.md              ← 你正在看的
├── SKILL.md               ← Claude Code Skill 定义（执行流程）
└── references/
    └── output-template.md ← 输出模板
```

**技术栈**：Playwright（浏览器自动化）→ JSON 数据采集 → LLM 提炼 → Markdown 报告

**为什么用 Playwright 而不是爬虫？** 传统爬虫只拿 HTML，看不到 JavaScript 渲染的内容——而很多平台（X、PH）的内容全靠 JS 动态加载。Playwright 打开真实浏览器，看到的是你肉眼看到的页面。详见[这篇文章]()中的讨论。

## 已知局限

- **AI 爬虫的盲区**：GPTBot、ClaudeBot 等不运行 JavaScript。如果你的输出发布在网上，AI 搜索可能看不到 JS 渲染部分。
- **语义相似 ≠ 质量**：LLM 在提炼时同样受"找最像的、不找最好的"这个结构性问题影响。报告中的排序不代表质量排序。
- **X 懒加载**：滚动加载最多抓到 7-8 条推文，无法突破 X 前端限制。
- **登录态过期**：各平台 cookie 有效期不同。X 通常数周到数月，雪球较短。发现采集数据异常时，重新跑第二步的登录命令。

## License

MIT
