# 学习通作业助手 · Skill 包

一个基于 **Playwright MCP** 的学习通（超星）作业自动化 skill：打开课程 → 找到未交作业 → 读题 → 填答案 → 保存校验。

**提交永远由你自己点。** 本包不含任何账号、密码、学校或课程信息。

---

## 包里有什么

```
chaoxing-homework/
└── SKILL.md      ← 全部内容（安装 + 首次任务 + 选择器 + 坑表 + 排障）
README.md         ← 你正在看的这份
```

只有两个文件，没有脚本、没有依赖、没有可执行文件。

---

## 一、装之前先确认

```bash
node -v      # 需要 18 或更高
```

再装浏览器内核（最容易漏的一步）：

```bash
npx playwright install chromium
```

---

## 二、装这个 skill

### 方式 A：作为 skill 使用（推荐，支持自动触发）

把 `chaoxing-homework` 整个文件夹放进你的 skill 目录：

| 客户端 | 用户级 skill 目录 |
|---|---|
| WorkBuddy | `~/.workbuddy/skills/` |
| Claude Code | `~/.claude/skills/` |

放好后重启客户端。之后说「帮我做学习通的作业」就会自动加载。

### 方式 B：只当手册看

不装 skill，直接打开 `SKILL.md`，把里面的步骤和代码块交给任意支持 MCP 的助手执行。

---

## 三、配 MCP（必做）

skill 本身不会操作浏览器，它依赖 Playwright MCP。在你的 MCP 配置文件里加入：

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

配置文件位置：

- WorkBuddy：`~/.workbuddy/mcp.json`
- Claude Desktop：`claude_desktop_config.json`
- Claude Code CLI：改用命令行
  ```bash
  claude mcp add playwright npx "@playwright/mcp@latest" --user-data-dir=/Users/你的用户名/.pw-mcp-profile
  ```

`--user-data-dir` 是重点：固定 profile 目录，**扫码登录一次长期有效**。不写它，每次任务都要重新扫码。

⚠️ 路径必须写**绝对路径**：Windows 用 `C:\Users\你的用户名\.pw-mcp-profile`，macOS / Linux 用 `/Users/你的用户名/.pw-mcp-profile`。写 `~` 在 Windows 上不会展开，会建出一个名叫 `~` 的目录，登录态照样丢。

改完配置**重启客户端**（或到连接器设置里点「信任」），否则工具不出现。

---

## 四、验证装好了

让助手列一下可用工具，能看到这五个就成功：

```
browser_navigate   browser_snapshot   browser_evaluate
browser_click      browser_tabs
```

看不到 → JSON 有语法错，或客户端没重启。

---

## 五、第一次任务怎么开始

直接对助手说：

```
帮我做学习通的作业，课程是《XXX》，今天做第二次课后作业
```

然后按提示扫码登录即可。助手会自己完成：读课程列表 → 进作业 → 抓题 → 填答案 → 暂时保存 → 刷新校验，最后把作答页留着让你核对。

**最后一步「提交」要你自己点。**

---

## 六、边界

| 做 | 不做 |
|---|---|
| 打开页面、进课程、抓题、填答案 | 代你登录（凭证不出你手） |
| 保存草稿、刷新校验 | 代你点提交 |
| 客观题给答案 | 思政/价值观论述题写成文（只给思路） |

仅限用于**你自己的账号**。把它用在别人的账号上属于越权，也与本包设计意图无关。

---

## 七、常见问题

**Q：每次都要扫码？**
没配 `--user-data-dir`，或配置改了没重启。

**Q：浏览器起不来 / 报找不到浏览器？**
```bash
npx playwright install chromium
```

**Q：填完了但刷新后答案没了？**
保存后没有等 AJAX 完成。重新填，点「暂时保存」后等 2 秒再刷新。

**Q：文本题填不进去？**
文本区是 UEditor，必须用 `UE.getEditor(id).setContent()`，改 DOM 无效。

**Q：能不能一次做多门课？**
可以，但每门课做完都要单独保存校验。助手每次都会先问清做哪门、哪份作业。

更多细节、全部选择器与 12 条坑，见 `chaoxing-homework/SKILL.md`。
