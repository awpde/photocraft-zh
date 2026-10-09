# PhotoCraft 简体中文版（Windows 构建）

自动从上游源码编译 **PhotoCraft** 的简体中文版 Windows 安装包，产出 MSI 与便携版 zip。

- 上游项目：<https://github.com/storytold/photocraft>（纯 Rust 重写的 Photoshop，Apache-2.0）
- 本仓库**不包含** PhotoCraft 的任何源码，只是构建流水线：每次构建都从上游现拉源码。

## ⚠️ 非官方构建声明

本仓库产出的是**非官方第三方构建**，**不是官方发布**，与原项目作者不存在隶属、
赞助或背书关系。

- 程序主体为原作者（PhotoCraft contributors）的作品，采用 **Apache-2.0** 许可
  （[LICENSE-APACHE](https://github.com/storytold/photocraft/blob/main/LICENSE-APACHE)，
  仓库内同时提供 [LICENSE-MIT](https://github.com/storytold/photocraft/blob/main/LICENSE-MIT)）。
- 本仓库**只修改构建配置与安装界面，未修改程序源码**（唯一改动是安装时可选安装位置）。
- 名称 `PhotoCraft` 仅用于说明本软件来源，不构成对其名称或商标的任何主张。
- 随每个 Release 一并分发的 `LICENSE-APACHE.txt`、`LICENSE-MIT.txt`、`NOTICE.txt`
  是上游许可证与声明**原文**；`UNOFFICIAL-BUILD-zh-CN.txt` 说明本构建具体改了什么。

## 下载

打开本仓库的 **Releases** 页面，下载带 `Latest` 标记的那个版本里的
`photocraft-<版本>-zh.1-windows-x64.msi`（或便携版 zip）。

## 这里做了什么改动

只有一处：**安装时可自选安装位置**。

上游的 `packaging/windows/photocraft.wxs` 本身没有任何安装界面定义（没有 `<UIRef>`、没有对话框），
安装包只有 Windows 最简进度界面，固定装到 `C:\Program Files\PhotoCraft`。本流水线通过两处**极小注入**
把 `overrides/installer-ui.wxs` 接上去，**不改动上游 wxs 的其余内容**（文件关联、注册表、组件都保持原样），
以免上游将来新增的东西被覆盖掉：

| 注入点 | 内容 |
|---|---|
| `photocraft.wxs` | 插入一行 `<UIRef Id="PhotocraftInstallerUI" />`（锚点命中数必须恰好 1，否则构建失败） |
| `package.ps1` | 把 `installer-ui.wxs` 加进 `wix build` 的源文件列表（原地替换，不增行） |

### 字体：与官方发版完全一致，且中文不需要嵌入字体

**中文字体：不嵌入，运行时读系统字体。** PhotoCraft 的界面中文字体来自系统
（Windows 上取 `C:\Windows\Fonts\msyh.ttc`，即微软雅黑），见上游 `crates/text/src/cjk.rs`。
界面西文主字体 `Inter` 则是程序自带的（`assets/fonts/`），与字体仓库无关。
所以中文显示正常、观感就是系统雅黑，**不需要为中文嵌入任何字体**。

（对照说明：姊妹项目 [pdfcraft-zh](https://github.com/awpde/pdfcraft-zh) 的上游没有这个系统字体机制，
所以那边**必须**嵌入 CJK 字体才能正常显示中文。）

**但日文字体仍按官方配置嵌入。** 本流水线检出官方 `release.yml` 锁定的同一个
`craft-fonts` commit（`abb83316…`，内容是日文字体）并设 `CRAFT_FONTS_REQUIRED=1`。
这样做的意义是：交付物与官方包**只差安装界面这一处**，不做无谓的偏离——
少了它日文界面会退回系统日文字体，体积小约 21 MB 但不再与官方一致。
（上游本身支持无字体构建，所以这是个可切换的取舍，不是硬性要求。）

界面译文（`crates/ui-egui/src/i18n/zh-hans.tsv`）由上游用 `include_str!` 编进二进制，
上游 **v0.3.0 起就自带简体中文**，所以只要上游有中文词条，构建出来的包就是中文界面。

## 构建流程里的防呆

任何一道不过都会让构建失败，而不是静默产出一个错的包：

1. **目标 ref 是否含中文词条** —— 没有就跳过构建（早期版本如 v0.2.0 不含）
2. **两处注入的锚点存在且唯一** —— 上游改了这两个文件会立刻报错，提示人工跟进
3. **检出的源码里中文词条内容正常** —— 行数与中文行数达标
4. **产出 MSI 里确实带上了自选安装位置界面** —— 字节查找对话框名与属性名
5. **便携版里确实带上了内嵌字体的许可文件** —— 即 `CRAFT_FONTS_DIR` 生效、与官方一致
6. **产出 exe 里确实含简体中文译文** —— 从同一次检出的词条里抽样，在 exe 里做字节查找

## 自动跟版

每天北京时间 06:00（UTC 22:00）自动检查上游最新 Release 并构建；也可以手动触发：

**Actions → 构建简体中文版 (Windows) → Run workflow**

- `upstream_ref`：默认 `main`（含中文词条）。填某个 tag 会构建那个 tag
- `version_suffix`：默认 `zh.1`，只用于文件名，避免与官方包混淆
- `make_release`：构建成功后是否创建 / 更新 Release

> GitHub 规则：公开仓库连续 60 天没有任何提交会**静默停用**定时任务。
> 本仓库平时不产生提交，所以定时运行时会自动检查，超过 50 天就补一个空提交防止被停用。

## 已知事项

- **未做代码签名**：上游用付费证书签名，本流水线没有，首次运行 Windows 会提示「未知发布者」，
  点「更多信息 → 仍要运行」即可。
- **覆盖升级**：MSI 使用 `MajorUpgrade AllowSameVersionUpgrades="yes"` 且 `UpgradeCode` 与官方一致，
  所以本包会**直接替换**已安装的官方同版本，不用先手动卸载。

## 许可、署名与改动声明

PhotoCraft 采用 **Apache-2.0**（仓库内同时提供 MIT 许可文本）。每个 Release 都附带：

| 文件 | 内容 |
|---|---|
| `LICENSE-APACHE.txt`、`LICENSE-MIT.txt` | 上游许可证全文 |
| `NOTICE.txt` | 上游版权、商标与第三方组件许可声明 |
| `UNOFFICIAL-BUILD-zh-CN.txt` | 本构建的改动说明（非官方声明）|

便携版 zip 内同时含 `LICENSE-MIT`、`LICENSE-APACHE`、`README.md` 与内嵌字体的许可文件。

界面中文字体使用系统字体（Windows 上为微软雅黑），因此本包**不分发任何中文字体**。

本构建为非官方第三方构建，与原项目作者无隶属、赞助或背书关系；名称 `PhotoCraft`
仅用于说明本软件来源。本仓库的流水线配置同样可按需取用。
