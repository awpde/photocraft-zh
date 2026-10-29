# AGENTS.md — 给接手维护本仓库的 agent / 人的第一份必读

> 这份文件里的规则**先于任何任务**生效。你可以在没有任何历史对话上下文的情况下，
> 只靠这个仓库独立完成维护。
>
> 深度内容在 [`docs/维护指南.md`](docs/)；这里只放「开工前必须知道的东西」。

---

## 0. 如果你是从聊天窗口接手（没有仓库工作目录）

按这个顺序读，**这些都是公开链接，不需要登录**：

1. 本文件
   <https://raw.githubusercontent.com/awpde/photocraft-zh/main/AGENTS.md>
2. 完整维护知识
   <https://raw.githubusercontent.com/awpde/photocraft-zh/main/docs/%E7%BB%B4%E6%8A%A4%E6%8C%87%E5%8D%97.md>
3. 姊妹仓库 PdfCraft 的入口
   <https://raw.githubusercontent.com/awpde/pdfcraft-zh/main/AGENTS.md>

> 仓库页面：<https://github.com/awpde/photocraft-zh>

**读完之后先别动手**：把理解的现状和打算做的事说出来，确认无误再执行。

---

## 1. 这个仓库是什么

**一句话**：这里没有任何 PhotoCraft 代码，只有一条 GitHub Actions 流水线 ——
拉上游 [storytold/photocraft](https://github.com/storytold/photocraft) 的源码，
用**极小改动**把上游安装向导的文案换成中文，编译成**简体中文 Windows 安装包**发布到 Releases。

| 要点 | 值 |
|---|---|
| 上游 | `storytold/photocraft`（MIT OR Apache-2.0，作者 storytold / ArtCraft） |
| 构建仓库 | `awpde/photocraft-zh`（本仓库） |
| 姊妹仓库 | `awpde/pdfcraft-zh`，**另有一份自己的 `docs/维护指南.md` 和 `AGENTS.md`** |
| 版本数 | **只有一个**（纯中文化）。PdfCraft 那个"含中文输入修复"的版本**在这里不存在也不需要** |
| 触发 | 每天 UTC 22:00 自动跟上游**最新 Release tag**；也可手动指定 ref |
| 产物 | MSI + 便携版 zip，每个包都带许可证与「非官方构建」声明四件套 |

**本仓库刻意做得比 PdfCraft 简单得多**，改动面越小，上游更新时越不容易坏：
`overrides/` 里**只有安装向导文案一个文件**，不改任何源码，不碰字体，不换图标。

---

## 2. 开工前，先按这个顺序读

1. **本文件**（你现在在读）
2. [`docs/维护指南.md`](docs/) —— 完整维护知识：注入方案、译文校验、踩坑集、风险表

然后**核查线上真实状态**（不要相信任何人的转述，包括本文件里的版本号）：

```bash
gh run list --repo awpde/photocraft-zh --limit 10
gh release list --repo awpde/photocraft-zh --limit 5
```

---

## 3. 铁律（违反了就是在制造事故）

### 3.1 永远不要只看绿对勾，必须回读构建日志

`gh run list` 显示 success **不代表**注入与校验真的过了。要在日志里逐条确认：

| 行 | 含义 |
|---|---|
| `✅ 已替换英文向导为 <UIRef Id="PhotocraftInstallerUI" />` | 界面注入生效 |
| `✅ 还原后与检出词条逐字节一致（N 字节，含简体中文）` | exe 里的译文完好 |
| `✅ 简体中文译文已以压缩形式编入 exe` | 中文确实编进去了 |
| `✅ InstallDirDlg` / `WIXUI_INSTALLDIR` / `DirectoryCombo` / `DirectoryListUp` + 中文 GBK 字节命中 | 上游原生目录页 + 中文向导文案都在 |

### 3.2 ★ 任何字符串前缀/比较判断必须用 `Ordinal`

**这是本仓库最隐蔽、代价最大的坑，已经让构建连续失败 4 次。**

上游词条文件在注释头之后有一个空行；Windows runner 检出为 CRLF → 空行变成单个 `\r`
（字节 `0x0D`）→ 被上游 build.rs「非词条行原样保留」打进二进制 → 我们的校验循环用
**culture-sensitive 的 `StartsWith`** 判断 `\x01` / `\x02` 前缀 —— 而 workflow 的 shell 是
**`pwsh`（.NET Core / ICU），ICU 排序规则把控制字符视为可忽略**，
`"\r".StartsWith("\x01")` 返回 **True** → 空行被误判成空译文条目 → 构建失败。

```powershell
# 错："\r".StartsWith([char]0x01)     ← pwsh 上是 True
# 对：
$line.StartsWith([char]0x01, [StringComparison]::Ordinal)
```

**本机是 PowerShell 5.1（NLS），同样写法返回 False —— 本地永远复现不出来。**
「本地过了 CI 挂了」时，第一反应就怀疑这个差异。

另外：跳过空行 / CR 行时**仍要把它原样追加到重建文本**，否则后续「逐字节比对源文件」会因为少行而失败。

### 3.3 界面注入的三个锚点必须恰好命中 1 次

锚点定义在 workflow 顶部的 `WXS_UI_ANCHOR` / `WXS_PROP_ANCHOR` / `PS1_ANCHOR`。
命中数 ≠ 1 时构建会**明确失败并打印命中次数** —— 这是上游改了 wxs 的可靠信号，
**不要为了让构建过去就去改锚点迁就**，先去看上游到底改了什么。

### 3.4 不要给这个仓库加字体处理

PhotoCraft 的界面中文字体是**运行时读系统字体**（Windows 上是微软雅黑，
`crates/text/src/cjk.rs`），不需要嵌入、不需要替换、不需要做字体清单校验。
`CRAFT_FONTS_REF` 锁的是**官方发版用的那个 commit**，只为让日文部分与官方一致。
**加字体处理 = 白干 + 多 8 MB 体积 + 多 4 个会失败���步骤。**

---

## 4. 本机 / 仓库的环境事实（踩过的坑）

| 事实 | 说明 |
|---|---|
| **这台机器上没有 git，也没有 GNU patch** | 需要做补丁类工作时，用姊妹仓库 `awpde/pdfcraft-zh` 里的 `tools/minipatch.py`（纯 Python） |
| `gh` 不在 PATH | 实际路径 `C:/Users/90714/tools/gh/bin/gh.exe`，已登录 `awpde`（scopes: repo + workflow） |
| **PowerShell 里 `gh api --jq '...'` 会被拆错参数** | 用 Python 走 curl + JSON 解析代替 |
| 没有 Rust | 本机不能编译验证，一切编译验证靠 CI |
| Python | `D:/Python/python.exe`（3.13） |
| 推文件**不要用 `git push`** | 用 `gh api --method PUT …/contents/<path>`（更新已有文件必须带 `sha`） |
| **workflow 的 shell 是 `pwsh`（.NET/ICU），本机是 PowerShell 5.1（NLS）** | 见 §3.2，这是本仓库的头号坑 |
| 含中文的 `.ps1` 文件必须加 UTF-8 BOM | 否则 5.1 按 ANSI 解码报莫名其妙的语法错误。workflow 内联的 pwsh 脚本不受影响 |
| 做字符串锚点匹配前先归一化行尾 | 远端文件可能是 CRLF，否则锚点命中 0 次，报出「已被他人改动」的误判 |

更完整的踩坑集见 `docs/维护指南.md` §7。

---

## 5. 用户是谁，以及怎么跟他沟通

用户**不写代码、不碰编译**，只下载安装包。他不熟 GitHub 网页。

- **下载链接必须是可直接点开的完整 URL**（https://github.com/awpde/photocraft-zh/releases 认 Latest 徽标，或 asset 直链）
- **不要主动生成交接文档**；维护知识就以 `docs/维护指南.md` 为准
- 大文件给直链让他浏览器自己下（本机下载很慢）
- 删旧 Release 前**先问**；临时目录用完问是否清理
- 他会收到 GitHub 的构建失败邮件并来问 —— **定时构建失败要当回事**
- 合规四件套必须随包；改动声明里要写明「界面中文字体使用系统字体（Windows 上为微软雅黑），因此本包不分发中文字体」

---

## 6. 不要做的事

- 不要只看构建是绿的就宣布完成（见 §3.1）
- 不要给这个仓库加字体处理、加图标替换、加源码补丁（见 §3.4）
- 不要为了让构建过去就改注入锚点（见 §3.3）
- 不要在任何字符串比较里省略 `[StringComparison]::Ordinal`（见 §3.2）