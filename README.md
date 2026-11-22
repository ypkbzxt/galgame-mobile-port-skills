# Galgame Mobile Port Skills

把 **PC 视觉小说（galgame）** 移植成 **安卓单 APK**（点开即玩）的两套可复用技能（Skill）。

这个仓库只包含**方法论 + 参考代码骨架**，**不包含任何游戏资源**。技能用于把**你已合法拥有的游戏文件**打包到你自己的手机上。

## 包含的技能

| 技能 | 说明 | 启动速度 | APK 体积 |
|---|---|---|---|
| [`artemis-tyranor-port`](artemis-tyranor-port/SKILL.md) | 基于 [Tyranor-Next](https://github.com/Weiss-UltimateSavior/Tyranor-Next) 定制：游戏数据内置 assets、首次启动解压、点开直达 | ~30 秒（首次解压） | 约 2GB |
| [`artemis-artroid-port`](artemis-artroid-port/SKILL.md) | 官方 Artemis Android 运行时（`com.ies_net.artemis` 壳）+ 解包后的游戏数据 | **~0.5 秒** | 约 1.4GB |

## 适用的游戏结构

先判断你的游戏是什么引擎（看游戏根目录文件）：

| 根目录文件特征 | 引擎 | 本仓库适用性 |
|---|---|---|
| `root.pfs` + `root.pfs.000/.001/...` | **Artemis Engine**（`pf8`/`pf6` magic） | ✅ 两个技能都适用 |
| 单独 `root.pfs` | **Artemis Engine** | ✅ 两个技能都适用 |
| `data.xp3` / `*.xp3` | KiriKiri / KrKr2 | ❌ 请用 Kirikiroid2，本仓库不涉及 |
| `Scene.pck` + `Gameexe.dat` | SiglusEngine | ❌ 请用 Tyranor 的 Siglus 宿主 |
| `game.rpa` / `*.rpy` | Ren'Py | ❌ 请用 Ren'Py 官方安卓打包 |

其他「封包引擎 + Android 官方运行时」的组合可以借鉴本仓库的思路，但需要自行适配对应引擎的运行时。

## 使用方式

1. 阅读对应技能目录下的 `SKILL.md`
2. 按「识别结构」确认你的游戏可用
3. 按「详细步骤」准备环境并构建
4. 按「关键坑」排查问题

技能格式遵循常见 AI Skill 约定（YAML frontmatter + Markdown 正文），可直接放入 DSH / Claude 等支持技能目录的 `skills/` 下使用，也可作为人工操作手册。

## 前提与限制

- **仅限个人使用**：把你自己合法拥有的游戏复制/打包到你自己拥有的设备上。
- **不提供游戏资源**：本仓库不含任何游戏文件、存档或授权补丁。
- **不绕过 DRM**：技能不包含破解、去授权、绕过正版验证的步骤。
- **不做分发**：请勿用本仓库的产物分发游戏给他人。

## 许可证

代码与文档采用 [MIT License](LICENSE)。

## 第三方来源

见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
