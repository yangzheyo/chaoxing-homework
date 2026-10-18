# 学习通作业自动化 · 分享包

一句话：让 AI（WorkBuddy / 腾讯 CodeBuddy 等支持 MCP 的助手）自动帮你做超星学习通的作业——打开网页 → 读题 → 出答案 → 批量填 → 暂时保存，最后你自己点提交。

> 前提认知：这是「AI 代填」工具，不是「代考」。它不替你登录、不替你点提交，提交这最后一步永远由你本人做（这是唯一能证明「本人在做」的环节）。

---

## 一、你需要装什么

| 依赖 | 干什么 | 怎么装 |
|---|---|---|
| Node.js ≥ 18 | 跑 Playwright 的运行时 | 去 https://nodejs.org 装 LTS 版，装完命令行里能敲 `npx -v` |
| 一个支持 MCP 的 AI 客户端 | 承载 skill + 连接器的宿主 | 你正在用的那个即可（WorkBuddy / CodeBuddy / Claude Desktop 等） |
| Playwright 浏览器内核 | AI 操控网页的浏览器 | 首次运行会自动装（见「四、首次运行」） |

---

## 二、三步装好

### 第 1 步 · 装 Node.js

- Windows：官网下载 LTS 安装包，一路下一步，勾选「Add to PATH」。
- 验证：命令行敲 `npx -v`，能输出版本号就 OK。

### 第 2 步 · 放 skill

把本文件夹里的 `chaoxing-homework` 整个目录，拷到你的 skills 目录：

- Windows：`C:\Users\<你的用户名>\.workbuddy\skills\`
- macOS / Linux：`~/.workbuddy/skills/`

拷完后**重启客户端**（或重载），AI 就能识别到这个 skill。

### 第 3 步 · 配 MCP 连接器

1. 打开你的 MCP 配置文件 `C:\Users\<你的用户名>\.workbuddy\mcp.json`（没有就新建）。
2. 把本文件夹里 `mcp-playwright.json` 的内容合并进 `mcpServers`。
3. **改路径**：把配置里的 `--user-data-dir=C:\Users\<你的用户名>\.pw_mcp_profile` 换成你自己的用户名（这是登录态存放位置，固定用你自己的）。
4. 改完后，在客户端的「连接器 / MCP」管理页找到 `playwright`，点「信任 / 启用」。

---

## 三、怎么用

对 AI 说一句话触发：

> 帮我做学习通作业。

然后按 AI 的引导走：

1. 它会打开学习通，**没登录就等你扫码/输密码**（登录这步你本人做）。
2. 登好它会读你的课程列表，**问你这轮做哪门课**。
3. 你说课名，它自动点进去 → 找「未交」作业 → 进作答页 → 读题 → 出答案 → 批量填 → 暂时保存。
4. 做完它喊你去浏览器检查，**你自己点「提交」**。

---

## 四、首次运行

第一次跑时，如果 AI 报「找不到浏览器 / browser not found」，在命令行执行一次：

```bash
npx playwright install chromium
```

装好浏览器内核后重试即可。之后就不用管了。

---

## 五、注意事项

- **登录态**：`--user-data-dir` 指定的那个文件夹（`.pw_mcp_profile`）保存登录态，登一次长期有效，别删。
- **换账号**：对 AI 说「换账号」，它会先登出再让你重新登录。
- **跑完会留临时文件**：AI 每次运行会在工作目录生成 `.playwright-mcp` 快照，定期清一下即可（不占多少空间）。
- **适用范围**：通识课 / 水课作业最稳；有严格标答的专业课，建议你提供教材答案让 AI 对照录入，准确率最高。

---

## 六、本包包含

```
├── README.md            ← 本说明（先看这个）
├── chaoxing-homework/   ← skill 执行手册（拷到 skills 目录）
│   └── SKILL.md
└── mcp-playwright.json  ← MCP 连接器配置（合并进你的 mcp.json）
```
