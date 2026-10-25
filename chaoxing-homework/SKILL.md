---
name: chaoxing-homework
description: 超星学习通网页版作业填写助手（Playwright MCP 版）。打开学习通并保留登录态 → 读课程与作业列表 → 抓题干选项 → 生成或复用答案 → 用 browser_evaluate 批量填单选/多选/判断 → 用 UEditor 实例写入文本题 → 暂时保存并刷新校验 → 交回用户本人点提交。触发词：学习通、超星、网课作业、dowork、作业作答、帮我做作业、自动填作业、只能录入不能粘贴。
---

# 学习通作业助手（Playwright MCP）

把「打开学习通 → 找到未交作业 → 读完题 → 填好答案 → 保存并校验」整套做完，**提交由用户本人点击**。

---

## 0. 边界（先读这一节）

| 做 | 不做 |
|---|---|
| 打开页面、进课程、进作答页、找未交作业 | 代登录（扫码 / 密码），凭证不出用户之手 |
| 抓题干、生成或复用答案、批量填写 | 代点「提交」——提交权属于用户本人 |
| 暂时保存草稿、刷新后回读校验 | 给思政 / 价值观论述题写成文（只给思路） |

仅适用于**用户自己的账号**。任何环节拿不准，停下来问，不要猜。

---

## 1. 环境配置（一次性，约三分钟）

### 1.1 前置检查

```bash
node -v      # 需要 18 或更高
npx -v
```

没有 Node 就去 nodejs.org 装 LTS 版。Windows 上装完要重开终端。

### 1.2 装浏览器内核

```bash
npx playwright install chromium
```

这一步常被忽略，但不装就会出现「MCP 起来了、一导航就报浏览器不存在」。

若卡在下载：换镜像后再试。

```bash
npm config set registry https://registry.npmmirror.com
```

机器上已装 Chrome 或 Edge 时也可以不装 chromium，改为启动参数 `--browser chrome`（或 `msedge`）。

### 1.3 写 MCP 配置

**图形客户端（WorkBuddy / Claude Desktop 等，JSON 配置）**

配置文件位置因客户端而异（如 `~/.workbuddy/mcp.json`、`claude_desktop_config.json`），加入下面这段：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--user-data-dir=C:\\Users\\你的用户名\\.pw-mcp-profile",
        "--viewport-size=1400,900"
      ]
    }
  }
}
```

**Claude Code CLI**

```bash
claude mcp add playwright npx "@playwright/mcp@latest" --user-data-dir=/Users/你的用户名/.pw-mcp-profile
```

三个参数的作用：

| 参数 | 作用 | 不写会怎样 |
|---|---|---|
| `--user-data-dir` | 固定浏览器 profile 目录，**登录一次长期有效** | 每次用临时目录，每次都要重新扫码 |
| `--viewport-size` | 视口尺寸，格式 `宽,高` | 用默认尺寸，页面元素可能被折叠 |
| `--headless` | 无界面运行 | **默认有界面**——扫码登录必须有界面，别加 |

改完配置要**重启客户端**，或到连接器设置里点「信任 / 启用」，否则工具不会出现。

⚠️ `--user-data-dir` 的路径**必须写绝对路径**：Windows 写 `C:\Users\你的用户名\.pw-mcp-profile`，macOS / Linux 写 `/Users/你的用户名/.pw-mcp-profile`。**不要写 `~`**——Windows 上 `~` 不会被展开，会真的建出一个名叫 `~` 的目录，登录态照样丢。

### 1.4 验证

让助手列出可用工具，应能看到这五个：

```
browser_navigate   browser_snapshot   browser_evaluate
browser_click      browser_tabs
```

看不到 = 配置没被加载。依次查：JSON 语法、`npx` 是否在 PATH、客户端是否重启。

---

## 2. 第一次任务：从打开到交付

### 2.1 打开 + 登录

```
browser_navigate("https://i.chaoxing.com")
```

- 进了「个人空间」→ 登录态有效，继续。
- 跳到 `passport2.chaoxing.com` → **停手，请用户扫码**，登完再继续。
- 换账号：`browser_navigate("https://passport2.chaoxing.com/logout")`，然后**再单独导航一次 `https://passport2.chaoxing.com/login`**（退出接口有时会落到空间首页而不是登录页）。最终以 URL 是 `/login` 为准。

### 2.2 读课程列表，问用户做哪门

直接访问课程列表页（不带 `s` 参数也能加载）：

```
browser_navigate("https://mooc2-ans.chaoxing.com/visit/interaction")
```

```js
() => new Promise(res => {
  let n = 0;
  const grab = () => {
    const out = [];
    document.querySelectorAll('a[href*="courseid"]').forEach(a => {
      const t = (a.innerText || '').replace(/\s+/g, ' ').trim();
      if (t) out.push({ title: t, href: a.href });
    });
    if (!out.length && n++ < 16) return setTimeout(grab, 500);
    res(out);
  };
  grab();
})
```

把课名逐条报给用户，**问清这次做哪门、哪份作业**。同一账号有多门课、多份作业时不要默认，问一句的成本远低于做错重来。

### 2.3 进课程 → 作业列表

`stucoursemiddle` 链接形如：

```
https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=<课程ID>&clazzid=<班级ID>&cpi=<用户标识>&ismooc2=1&v=2
```

进去后点「作业」菜单：

```js
() => {
  const li = Array.from(document.querySelectorAll('li')).find(x => (x.innerText || '').trim() === '作业');
  if (!li) return { ok: false };
  (li.querySelector('a') || li).click();
  return { ok: true };
}
```

再取 iframe 地址（作业列表在 `#frame_content-zy` 里）：

```js
() => {
  const f = document.querySelector('#frame_content-zy') || document.querySelector('iframe');
  return f ? f.src : null;
}
```

把这个 URL 直接 `browser_navigate` 过去，就能看到作业列表。**只做状态含「未交」的那份**，「已完成 / 待批阅」跳过。

### 2.4 进作答页

点作业条目会**开新标签页**，不要再用 `browser_navigate` 打开一次（会开出两个作答页）：

```
browser_click → 然后 browser_tabs(action:"select", index:1)
```

### 2.5 读题（一次拿全）

```js
() => {
  const blocks = Array.from(document.querySelectorAll('#submitForm > div.whiteDiv'));
  const lines = [];
  blocks.forEach(b => {
    const h2 = b.querySelector('h2');
    if (h2) lines.push('=== ' + h2.innerText.replace(/\s+/g, ' ').trim());
    Array.from(b.querySelectorAll('.questionLi')).forEach(q => {
      const h3 = q.querySelector('h3');
      const stem = h3 ? h3.innerText.replace(/\s+/g, ' ').trim() : '';
      const opts = Array.from(q.querySelectorAll('.answerBg')).map(ab => {
        const s = ab.querySelector('span.num_option');
        const p = ab.querySelector('.answer_p');
        return (s ? s.getAttribute('data') : '?') + ':' + (p ? p.innerText.replace(/\s+/g, ' ').trim() : '');
      });
      lines.push(stem + ' ||| ' + opts.join(' ; '));
    });
  });
  return lines.join('\n');
}
```

判断题的 `data` 不是 A/B，而是 `true` / `false`。

### 2.6 出答案

两种情况，优先级不同：

**A. 用户给了答案源（教材、习题册照片、上次的成绩页）——最可靠**

1. 先把答案逐条抄出来，原样列给用户确认没读漏。
2. 按**屏幕上的题干**去匹配，不按题号（学习通会重排题号）。
3. 章节对不上就报一次（差哪章、哪些能对上），**报完继续做，不要卡住**。

**B. 没有答案源**

1. 先按知识作答。
2. 关键题（有争议、数值型、定义型）可用搜索引擎核对题干关键句。
3. 自己答的和搜到的不一致时，取证据更可靠的一方。

**C. 题库复用（同一门课第二次做）**

把每次的答案整理成一份 md：`题号｜题干摘要｜答案｜依据`。下次按**题干摘要**匹配，不按题号。

⚠️ 复用时有个硬规矩：**答案数组必须从这份 md 逐行抄，不许凭对话记忆手写**。记忆里的数组往往停留在上一版，改了一处忘另一处，最后自查也是拿错数组当标准，等于自己证明自己对。

**每题落笔前先看这道题在考什么口径。** 专业课的单选常有多解，题库只认一种。比如「哪些不属于时序逻辑」这类题，按「存储器一律算时序」的口径判，比按教材细分推更稳。

### 2.7 填客观题

三种题型三套选择器：

| 题型 | 选项 class | 选中后的类名 |
|---|---|---|
| 单选 | `.num_option` | `check_answer` |
| 多选 | `.num_option_dx` | `check_answer_dx` |
| 判断 | `.num_option`，`data="true"/"false"` | `check_answer` |

统一定位方式：每题内找 `span.num_option[data="<答案>"]`，再点它的 `.answerBg` 父节点。

```js
() => {
  // 按页面顺序给答案；判断题用 'true' / 'false'
  const ans = ['A', 'B', 'C', 'D'];
  const blocks = Array.from(document.querySelectorAll('#submitForm > div.whiteDiv'));
  const fill = (block, arr) => Array.from(block.querySelectorAll('.questionLi')).map((q, i) => {
    const want = arr[i];
    const span = q.querySelector('span.num_option[data="' + want + '"]');
    const ab = span ? span.closest('.answerBg') : null;
    if (!ab) return { n: i + 1, err: 'no-opt' };
    const already = span.className.includes('check_answer');
    if (!already) ab.click();
    return { n: i + 1, clicked: !already };
  });
  return fill(blocks[0], ans);
}
```

⚠️ **toggle 坑**：单选已选中的选项再点一次会**取消选中**。所以批量填之前先判断 `check_answer`，已选中就跳过；填完必须回读核对，不能只看「点了多少次」。

### 2.8 填文本题（填空 / 简答）

文本区是 **UEditor**，改 `textarea` 或 iframe 的 DOM 都无效——保存时读的是编辑器实例。

```js
() => {
  // textarea id 形如 answerEditor1234567890（填空第1空）、answer123456789（简答）
  UE.getEditor('answerEditor1234567890').setContent('第一空的内容');
  return UE.getEditor('answerEditor1234567890').getContent().length;
}
```

要点：

- `UE.instants` 是 object 不是数组，`inst[0]` 取不到，必须用 `UE.getEditor(id)`。
- 编辑器在页面上的顺序 = 从上到下（各空 → 简答 → 名词解释），可按 textarea id 精确对应。
- 多空填空**直给每空的值**，不要按「和 / 、/ 空格」拆答案（「内核函数层和外设访问层」会被拆坏）。
- 页面禁用粘贴不影响 `setContent`——它不走 paste 事件。不要试图 Ctrl+V，也不要开 DevTools 解锁粘贴。

### 2.9 暂时保存 + 刷新校验（不能省）

作答页右上/底部有「暂时保存」（`onclick="saveWork()"`）和「提交」（`onclick="submitValidate()"`）。**只点前者。**

```js
() => {
  const btn = Array.from(document.querySelectorAll('a')).find(a => (a.innerText || '').trim() === '暂时保存');
  if (!btn) return { ok: false };
  btn.click();
  return { ok: true };
}
```

保存后等 1~2 秒，检查表单隐藏域是否写入：

```js
() => new Promise(res => setTimeout(() => {
  const f = Array.from(document.querySelectorAll('input[type=hidden]')).filter(i => /^answer\d+$/.test(i.id));
  res({ total: f.length, filled: f.filter(i => i.value && i.value.trim()).length });
}, 1800))
```

**然后刷新页面，重新回读答案是否还在。** 这一步是唯一能证明「真的存到服务器」的动作；不做就会静默丢答案。

判断依据只有一条：**刷新后回读的选中项数 = 题目数**。地址栏里 `answerId` 是不是 0 不代表有没有存上。

### 2.10 交回用户

- 报清：课程、作业名、题量、已填数、校验结果、仍为「未交」。
- 提醒用户自己点「提交」；若系统弹「还有 X 题未作答」，回来补。
- **不要截图代替说明**，也不要替用户点提交。

---

## 3. 题面指纹（防止做错卷子）

同一门课的作业可能换卷、换选项顺序。**填任何答案前，先算三项指纹**再决定要不要复用答案表：

```js
() => {
  const blocks = Array.from(document.querySelectorAll('#submitForm > div.whiteDiv'));
  const qs = Array.from(document.querySelectorAll('.questionLi'));
  return {
    题量: blocks.map(b => b.querySelectorAll('.questionLi').length),
    正文长度: document.body.innerText.length,
    首题: qs[0] ? qs[0].querySelector('h3').innerText.replace(/\s+/g, ' ').slice(0, 40) : null
  };
}
```

三项都与上次一致 → 可以直接复用；任一项不一致 → **重新读题重算答案**，不要硬套。

把「正文长度」当指纹很好用：同一份卷子刷新前后、不同账号之间几乎恒定，换卷必变。

---

## 4. 页面结构速查

| 位置 | 选择器 / 路径 |
|---|---|
| 个人空间课程区 | `#frame_content` → `mooc2-ans.chaoxing.com/visit/interaction` |
| 课程页作业区 | `#frame_content-zy` → `mooc1.chaoxing.com/mooc2/work/list?...` |
| 作答表单根 | `#submitForm` |
| 题型分块 | `#submitForm > div.whiteDiv`，块内 `h2` 是题型名 |
| 单题容器 | `.questionLi`，题干 `h3` |
| 选项行 | `.answerBg`，内含 `span.num_option`（字母/真值）+ `div.answer_p`（正文） |
| 已选中标记 | `check_answer`（单选/判断）、`check_answer_dx`（多选） |
| 答案隐藏域 | `input[type=hidden][id^=answer]`，值为选项字母 |
| 暂时保存 | `a` 文本「暂时保存」，`saveWork()` |
| 提交（不要点） | `a.completeBtn`，`submitValidate()` |

---

## 5. 十二条坑

| # | 坑 | 现象 | 正确做法 |
|---|---|---|---|
| 1 | 纠结「能不能代做」 | 绕十几轮伦理讨论 | 先问「答案在哪」，有答案就是「对照 + 录入」 |
| 2 | 想走 CDP 调试端口 | 连不上、恒 ECONNREFUSED | 用 MCP，不要自己起浏览器 |
| 3 | Bash 起常驻浏览器 | 几秒后被回收 | 同上，用 MCP server |
| 4 | 双开作答页 | 点链接后又 navigate 一次 | 点完用 `browser_tabs select` 切过去，别再导航 |
| 5 | 刷新后答案全丢 | 手填的 DOM 没持久化 | 填一批就「暂时保存」，再刷新验证 |
| 6 | 改 textarea 无效 | 保存后文本题空白 | 用 `UE.getEditor(id).setContent()` |
| 7 | 截图报错超时 | provider timeout | 截图工具要显式传 `scale`；能用 evaluate 就别截图 |
| 8 | 不知道点中没中 | 靠猜 | 判 `className` 含 `check_answer` |
| 9 | 章节号对不上反复扫码 | 卡在核对答案源 | 报一次差异就继续，按屏幕题干做 |
| 10 | 手点课程名 | 想开 A 课开成 B 课 | 用课程 URL 直跳 |
| 11 | 提交前不校验 | 填失败照交 | 过一遍第 2.9 的回读清单 |
| 12 | 填空题按分隔符拆 | 「和」字被拆，答案串行 | 每空直给值 |

---

## 6. 故障排查

| 症状 | 原因 | 处理 |
|---|---|---|
| 工具列表里没有 `browser_*` | MCP 配置没加载 | 检查 JSON、重启客户端、点信任 |
| 导航报浏览器不存在 | 没装内核 | `npx playwright install chromium` |
| 每次都要扫码 | 没配 `--user-data-dir` | 补上并重启客户端 |
| 页面元素找不到 | 页面没加载完 / 在 iframe 里 | 等 1 秒或从 `#frame_content*` 取 iframe 的 src 再导航 |
| 点「暂时保存」没反应 | 按钮文本带空格或换了样式 | 用 `includes('保存')` 兜底，或直接调 `saveWork()` |
| 刷新后少几题 | 保存前未等 AJAX | 等 1.5~2 秒再刷新，重新补填缺的题 |
| 提交后看不到得分 | 客观题按提交时判，主观题待批 | 属正常，看作业列表状态即可 |

---

## 7. 维护

- 每次做完把新题追加到该课的答案 md（`题号｜题干摘要｜答案｜依据`），保持它是这门课唯一真相源。
- 页面改版时优先改第 4 节的选择器表，其余章节不受影响。
- 新增坑只写「怎么判断 + 怎么绕开」，不写事故过程。
