---
name: daily-scan
description: 扫描 X、Reddit、雪球当日热点，提炼 AI+科技趋势、产品机会，输出结构化 Markdown 报告
allowed-tools: Bash(playwright-cli:*) Bash(mkdir:*) Bash(sleep:*) Bash(cat:*) Bash(echo:*) Read Write Edit Glob Grep
---

# Daily Scan — X + Reddit + 雪球 趋势扫描

> 用 Playwright 持久化 profile 采集三平台当日信息 → LLM 提炼 → 写入 `output/YYYY-MM-DD.md`

## 前置条件

- Playwright CLI 已安装（`npm install -g @playwright/cli`）
- Edge 浏览器可用
- 持久化 profile 在 `$HOME/.claude/playwright-edge-profile/`（X、Reddit、雪球均已登录）
- **首次使用？** 按照 README.md 第二步逐平台登录，保存 cookie 到 profile

## 铁律（不可跳）

1. **所有网站一律有头模式。**
2. **优雅关闭**：禁止系统级杀 Edge 进程。只用 `playwright-cli close`。
3. **登录验证**：每次打开平台后必须验证登录态，未登录立即报错，不继续。
4. **Profile 串行**：同一 profile 同时只能一个进程使用。
5. **每条内容必须有原文链接**：所有引用的信号、数据、观点都必须附带可点击的原始 URL。无链接 = 不可信 = 不发。
6. **禁止中途停止**：所有阶段必须全部执行。

---

## 执行流程

### 阶段 1：准备

```bash
mkdir -p output
mkdir -p $HOME/.claude/tmp
```

日期变量（北京时间 UTC+8）：
```bash
TODAY=$(TZ=Asia/Shanghai date +%Y-%m-%d)
echo "扫描日期: $TODAY"
```

### 阶段 1b：外网可达性预检

```bash
playwright-cli -s=net-check open "https://old.reddit.com/" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed 2>&1

sleep 2

playwright-cli -s=net-check eval "function() {
  var posts = document.querySelectorAll('.thing.link');
  return { loaded: posts.length > 0, count: posts.length };
}" 2>&1

playwright-cli -s=net-check close 2>&1
sleep 1
```

- ✅ `loaded=true, count >= 20` → 外网通，继续
- ⚠️ `loaded=false` 或超时 → **暂停！** 通知用户外网不通（X/Reddit 不可用）。是否继续仅雪球的缩减版扫描？

### 阶段 2：X 采集

```bash
playwright-cli -s=scan-x open "https://x.com/home" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed 2>&1

sleep 3

# 验证登录态
playwright-cli -s=scan-x eval "function() {
  var title = document.title;
  var onHome = window.location.href.includes('/home');
  var hasTimeline = !!document.querySelector('[data-testid=primaryColumn]');
  return { title: title, onHome: onHome, hasTimeline: hasTimeline };
}" 2>&1
# 预期：title 包含 "Home / X"，onHome=true，hasTimeline=true
# ❌ 如果 title 是 "X。尽是新鲜事" 或 onHome=false → 未登录，立即报错终止！
```

如果登录验证通过，继续采集：

```bash
# 滚动加载
playwright-cli -s=scan-x eval "function() {
  for (var i = 0; i < 8; i++) {
    window.scrollTo(0, document.body.scrollHeight);
  }
  return 'scrolled 8x, ' + document.querySelectorAll('[data-testid=tweet]').length + ' tweets';
}" 2>&1

sleep 3

# 抓取推文
playwright-cli -s=scan-x eval "function() {
  var tweets = Array.from(document.querySelectorAll('[data-testid=tweet]'));
  if (!tweets.length) {
    var articles = Array.from(document.querySelectorAll('article'));
    return { source: 'article-fallback', count: articles.length, tweets: articles.slice(0, 30).map(function(a) {
      return {
        text: (a.querySelector('[data-testid=tweetText]')?.textContent || a.textContent).slice(0, 300),
        author: (a.querySelector('[data-testid=User-Name]')?.textContent || '').slice(0, 50),
        time: a.querySelector('time')?.getAttribute('datetime') || '',
        url: (Array.from(a.querySelectorAll('a[href*=\"status\"]')).map(function(l){return l.href;})[0] || ''),
        likes: '', replies: '', retweets: ''
      };
    }).filter(function(t) { return t.text.length > 10; }) };
  }
  return { source: 'tweet-testid', count: tweets.length, tweets: tweets.slice(0, 30).map(function(t) {
    return {
      text: (t.querySelector('[data-testid=tweetText]')?.textContent || '').slice(0, 300),
      author: (t.querySelector('[data-testid=User-Name]')?.textContent || '').slice(0, 50),
      time: t.querySelector('time')?.getAttribute('datetime') || '',
      url: (Array.from(t.querySelectorAll('a[href*=\"status\"]')).map(function(l){return l.href;})[0] || ''),
      likes: t.querySelector('[data-testid=like]')?.getAttribute('aria-label') || '',
      replies: t.querySelector('[data-testid=reply]')?.getAttribute('aria-label') || '',
      retweets: t.querySelector('[data-testid=retweet]')?.getAttribute('aria-label') || ''
    };
  }).filter(function(t) { return t.text.length > 10; }) };
}" > $HOME/.claude/tmp/x-tweets.json

# 后处理：清洗输出 → 纯 JSON
node -e "
var fs=require('fs');
var raw=fs.readFileSync(process.env.HOME+'/.claude/tmp/x-tweets.json','utf8');
var m=raw.match(/\{[\s\S]*?\"tweets\"[\s\S]*?\}/);
if(!m){console.log('PARSE FAILED');process.exit(1);}
var d=JSON.parse(m[0]);
fs.writeFileSync(process.env.HOME+'/.claude/tmp/x-tweets-clean.json',JSON.stringify(d,null,2));
console.log('cleaned '+d.tweets.length+' tweets');
if(d.tweets.length<5) console.log('WARNING: low tweets, X lazy-load may limit');
"

playwright-cli -s=scan-x close 2>&1
sleep 1

playwright-cli list 2>&1 | grep -q "scan-x" && echo "ERROR: scan-x still running!" || echo "scan-x closed OK"
```

### 阶段 3：Reddit 采集

```bash
playwright-cli -s=scan-reddit open "https://old.reddit.com/" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed 2>&1

sleep 2

# 验证登录态
playwright-cli -s=scan-reddit eval "function() {
  var userLink = document.querySelector('#header-bottom-right .user a');
  var blocked = document.body.textContent.includes('blocked by network security');
  return { loggedIn: !!userLink, username: userLink?.textContent?.trim() || '', blocked: blocked };
}" 2>&1
# 预期：loggedIn=true, blocked=false
```

#### 3a：帖子列表

```bash
playwright-cli -s=scan-reddit eval "function() {
  var posts = Array.from(document.querySelectorAll('.thing.link'));
  return {
    count: posts.length,
    posts: posts.slice(0, 30).map(function(p) {
      return {
        subreddit: (p.querySelector('a.subreddit')?.textContent || '').trim(),
        title: (p.querySelector('a.title')?.textContent || '').trim(),
        score: (p.querySelector('.score.unvoted')?.textContent || p.querySelector('.score')?.textContent || '').trim(),
        comments: (p.querySelector('a.comments')?.textContent || '').trim().replace(/\\s.*/, ''),
        url: p.querySelector('a.title')?.getAttribute('href') || '',
        selftext: (p.querySelector('.expando .usertext-body .md')?.textContent || '').slice(0, 300)
      };
    }).filter(function(p) { return p.title.length > 5; })
  };
}" > $HOME/.claude/tmp/reddit-posts.json

node -e "
var fs=require('fs');
var raw=fs.readFileSync(process.env.HOME+'/.claude/tmp/reddit-posts.json','utf8');
var m=raw.match(/\{[\s\S]*?\"posts\"[\s\S]*?\}/);
if(!m){console.log('PARSE FAILED');process.exit(1);}
var d=JSON.parse(m[0]);
fs.writeFileSync(process.env.HOME+'/.claude/tmp/reddit-posts-clean.json',JSON.stringify(d,null,2));
console.log('cleaned '+d.posts.length+' posts');
"
```

#### 3b：评论区采样（仅高互动帖子）

从 `reddit-posts.json` 中选出 comments 数 ≥ 10 的帖子，逐个进入评论区抓热评：

```bash
playwright-cli -s=scan-reddit goto "https://old.reddit.com/POST_URL"
sleep 3

playwright-cli -s=scan-reddit eval "function() {
  var selectors = ['.usertext-body .md', '.entry .md', '.comment .md'];
  var comments = [];
  for (var s = 0; s < selectors.length && comments.length === 0; s++) {
    comments = Array.from(document.querySelectorAll(selectors[s]));
  }
  if (!comments.length) {
    var area = document.querySelector('.commentarea');
    return { source: 'commentarea-fallback', text: area ? area.textContent.slice(0, 2000) : 'NO COMMENTS FOUND' };
  }
  return { source: 'entries', count: comments.length, comments: comments.slice(0, 10).map(function(c) {
    return {
      text: (c.textContent || '').slice(0, 300),
      score: c.closest('.comment')?.querySelector('.score')?.textContent?.trim() || ''
    };
  }).filter(function(c) { return c.text.length > 10; }) };
}" > $HOME/.claude/tmp/reddit-comments-N.json
```

```bash
playwright-cli -s=scan-reddit close 2>&1
sleep 1

playwright-cli list 2>&1 | grep -q "scan-reddit" && echo "ERROR: scan-reddit still running!" || echo "scan-reddit closed OK"
```

> **为什么用 old.reddit.com**：新版 Reddit 的 `<shreddit-post>` web component 响应极慢。老版用 `.thing.link` 纯 DOM 选择器，秒级返回。

### 阶段 4：雪球采集

```bash
playwright-cli -s=scan-xueqiu open "https://xueqiu.com/" \
  --browser=msedge --profile=$HOME/.claude/playwright-edge-profile --persistent --headed 2>&1

sleep 3

# 验证登录态
playwright-cli -s=scan-xueqiu eval "function() {
  var isLogin = !document.body.textContent.includes('立即登录');
  return { title: document.title, isLogin: isLogin };
}" 2>&1
# 预期：isLogin=true，title 包含 "我的首页"
```

```bash
# 滚动加载
playwright-cli -s=scan-xueqiu eval "function() {
  for (var i = 1; i <= 8; i++) {
    window.scrollTo(0, document.body.scrollHeight);
  }
  return 'scrolled 8x, ' + document.querySelectorAll('article').length + ' articles on page';
}" 2>&1

sleep 3

# 抓取 raw text
playwright-cli -s=scan-xueqiu eval "function() {
  var articles = Array.from(document.querySelectorAll('article'));
  return articles.slice(0, 35).map(function(a) {
    return { text: (a.textContent || '').trim().slice(0, 600) };
  }).filter(function(p) { return p.text.length > 30; });
}" > $HOME/.claude/tmp/xueqiu-raw.json

node -e "
var fs=require('fs');
var raw=fs.readFileSync(process.env.HOME+'/.claude/tmp/xueqiu-raw.json','utf8');
var lines=raw.split('\n');
var start=lines.findIndex(function(l){return l.trim().startsWith('[')});
var end=lines.length-1;
while(end>start && !lines[end].trim().endsWith(']')) end--;
var json=lines.slice(start, end+1).join('\n');
var d=JSON.parse(json);
fs.writeFileSync(process.env.HOME+'/.claude/tmp/xueqiu-posts-clean.json', JSON.stringify(d,null,2));
var texts=d.map(function(p,i){return '['+i+'] '+p.text;}).join('\n\n---\n\n');
fs.writeFileSync(process.env.HOME+'/.claude/tmp/xueqiu-texts.txt', texts);
console.log('cleaned '+d.length+' articles');
"

playwright-cli -s=scan-xueqiu close 2>&1
sleep 1

playwright-cli list 2>&1 | grep -q "scan-xueqiu" && echo "ERROR!" || echo "xueqiu closed OK"
```

### 阶段 5：LLM 提炼

读取以下文件：

- `$HOME/.claude/tmp/x-tweets-clean.json`
- `$HOME/.claude/tmp/reddit-posts-clean.json`
- `$HOME/.claude/tmp/reddit-comments-*.json`（如有）
- `$HOME/.claude/tmp/xueqiu-posts-clean.json` 或 `$HOME/.claude/tmp/xueqiu-texts.txt`

**提炼维度：**

1. **🔥 今日热点（3-5 条）** — 跨平台共振的话题。标注热度（🟢高/🟡中/🔴低）和信号来源。
2. **💡 产品机会信号（2-4 条）** — 宏观方向级。Reddit/X 上的 "I wish there was a tool..." 和反复出现的痛点。
3. **🔧 微机会（2-4 条）** — 评论区中的功能/插件级诉求，一人可 build。
4. **📊 趋势指标** — 对比前几日扫描，话题升/降温情况。
5. **🗣️ 值得深读（3-5 条）** — 今日最有信息量的帖子 + 为什么值得看。

**提炼原则：**
- 宁可少报，不报噪音。每条信号必须能在原始数据中找到出处。
- 区分 "hype" 和 "真实需求"。
- **每条信号必须有原文链接。** 无链接 = 不可信 = 不写入。
- 标注每条信号的信心等级：🟢高（多源交叉验证）/ 🟡中（单源但信息量大）/ 🔴低（单次提及，待观察）。

### 阶段 6：写入 Markdown

按 `references/output-template.md` 格式，写入 `output/$TODAY.md`。

### 阶段 7：自检

```bash
TODAY=$(TZ=Asia/Shanghai date +%Y-%m-%d)
echo "=== 交付物检查 ==="
[ -f "output/$TODAY.md" ] && echo "✅ $TODAY.md" || echo "❌ 缺少 $TODAY.md"
```

---

## 故障手册

**playwright-cli JSON 污染**
- 现象：`eval` 输出到文件后，文件内容为 `### Result\n{JSON}\n### Ran Playwright code\n...`
- 修复：每个 `> file` 后紧跟 `node -e` 清洗步骤（正则提取纯 JSON），生成 `-clean.json`

**bash 特殊字符转义冲突**
- 现象：`$` 在 `"..."` 中被 bash 解释
- 修复：eval 代码用 `"function() { ... }"` 而非 `"(() => { ... })"`

**Windows bash fork 资源耗尽**
- 现象：循环中报 `fork: retry: Resource temporarily unavailable`
- 修复：多步操作合成一个 eval（在 JS 内部循环）

**Reddit 评论选择器**
- `.comment .md` → 空（实测）
- `.usertext-body .md` 或 `.entry .md` → 正确选择器
- fallback：`.commentarea` 全文抓取

**X 懒加载限制**
- 8 次滚动可能只拿到 7-8 条推文
- 属于 X 前端机制，无法突破。接受低采集量，不阻塞流程。

**profile 释放确认**
- `close` 后必须 `sleep 1` 再检查 `playwright-cli list`
- 不确认就直接开下个平台 → profile 冲突导致新浏览器白屏或崩溃

**登录态过期**
- X cookie 通常数周到数月；雪球较短
- 发现采集数据异常时，重新运行 README 第二步的登录命令
