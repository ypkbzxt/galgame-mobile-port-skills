---
name: kirikiri-krkr2-port
description: 把 KiriKiri2 / KiriKiriZ（.xp3）的 PC galgame 做成安卓可玩：优先用成熟运行器 Kirikiroid2（KRKR2）直接跑 .xp3；需要"点开即玩"时，用 KRKR2 源码定制成单 APK（游戏数据内置）。适用于《千恋万花》这类 KiriKiri2 游戏（判据见"识别结构"）。
---

# KiriKiri2 / KiriKiriZ 游戏 → 安卓（Kirikiroid2 方案）

## 这是干什么的

把 `.xp3` 结构的 PC galgame 做成安卓可玩：

- **方案 A（推荐）**：用现成运行器 **Kirikiroid2（KRKR2）**，把游戏目录放进手机即可玩——**不需要解包**，KRKR2 直接读 `.xp3`
- **方案 B**：需要"点开即玩"的单 APK 时，用 KRKR2 源码定制（引擎 + 游戏数据内置）

## 识别结构（先判断能不能用本方案）

看游戏根目录：

| 文件 | 引擎 | 说明 |
|---|---|---|
| `data.xp3` / `*.xp3` | **KiriKiri2 / KiriKiriZ** | ✅ 本方案 |
| `root.pfs`（pf8/pf6） | Artemis Engine | ❌ 用 artemis-* 技能 |
| `Scene.pck` | SiglusEngine | ❌ 用 Tyranor 的 Siglus 宿主 |
| `*.rpa` / `*.rpy` | Ren'Py | ❌ 用 Ren'Py 官方打包 |

**判据一句话**：目录里有 `.xp3` 文件就是 KiriKiri 系（`data.xp3` 通常最大，是主数据）。

## 方案 A：用现成 Kirikiroid2（最省事）

### 1. 获取 KRKR2

- 官方仓库：<https://github.com/zeas2/Kirikiroid2>（C++ 移植，含源码）
- 去广告构建：<https://github.com/enaix/Kirikiroid2-debloated>
- 补丁库（加密 xp3 用）：<https://github.com/zeas2/Kirikiroid2_patch>

APK 安装到手机（arm64）。

### 2. 准备游戏目录

- 把游戏 PC 目录（含 `data.xp3`、`*.xp3`、`*.ini`、`*.tjs` 等）整体复制到手机
- **路径纯英文**（`/sdcard/KrkrGames/<英文名>/`），KRKR2 对中文路径敏感
- 可参考社区惯例：游戏统一放 `KrkrGames/` 目录

### 3. 在 KRKR2 里选游戏

打开 KRKR2 → 选择游戏目录（或直接选 `data.xp3`）→ 启动。

### 4. 加密 xp3 的处理

**Yuzusoft（柚子社）等会社的 `.xp3` 是加密的**，原版 KRKR2 可能直接报错/黑屏。两条路：

1. **打补丁**：从 `Kirikiroid2_patch` 找对应游戏的补丁（按游戏名/会社），放进 KRKR2 的 `patch` 目录
2. **解包再封包**：用 `KrkrExtract`（<https://github.com/unlimit999/KrkrExtract>，支持含加密的 krkr2/krkrz xp3，含柚子社格式）把游戏解包成目录，KRKR2 可直接跑解包目录

> 如果目标是「自己的游戏文件 + 自己的手机」，社区常规做法是「解包 → 重打包为未加密 xp3」或「用 patch」。注意：**只处理你自己合法拥有的游戏**。

## 方案 B：单 APK 内置（点开即玩）

KRKR2 是**开源 C++ 工程**，可以照 `artemis-tyranor-port` 的思路定制：

1. 拉取 `zeas2/Kirikiroid2` 源码，用 NDK 编译
2. 游戏数据（`.xp3` 或解包目录）放进 `assets/`
3. 首次启动解压到 App 私有目录
4. 把启动入口改成「直接启动内置游戏」
5. 改包名/应用名/图标，签名出 APK

与 Tyranor 方案的关键差异：

| | KRKR2 定制 | Tyranor 定制 |
|---|---|---|
| 数据格式 | `.xp3` 原样或解包目录 | `root.pfs` 原样（首启解压） |
| 引擎来源 | 自带源码（C++ + Java） | 第三方多引擎宿主 |
| 工作量 | **大**（NDK 编译 + 改启动） | 中（只改 Kotlin 入口） |

**建议**：除非必须单 APK，否则优先方案 A——KRKR2 本身体验已经接近原生。

## 关键坑

1. **加密 xp3**：Yuzusoft 等会社的包默认加密，必须 patch 或解包，否则黑屏/报错。
2. **中文路径**：KRKR2 对路径敏感，游戏目录用英文。
3. **存档位置**：KRKR2 的存档通常在游戏目录下的 `savedata/` 或 App 私有目录；跨设备迁移前先确认。
4. **`.xp3` 不需要解包**：方案 A 直接读；只有加密包才需要解包/补丁。
5. **krkrz（KiriKiriZ）与 krkr2 差异**：新版游戏（KiriKiriZ）用 `.xp3` 但脚本格式不同，KRKR2 支持大部分，个别游戏需要 KrkrExtract 特殊处理。
6. **Android 15**：KRKR2 老版本可能在新系统上闪退，优先用较新的 debloated 构建。

## 可适配范围

- **所有 KiriKiri2 / KiriKiriZ 游戏**（`.xp3`）：换游戏只需把游戏目录放到手机
- 加密包：用对应 patch 或 `KrkrExtract` 解包
- **不适用**：Artemis（root.pfs）、Siglus、Ren'Py、Unity

## 与 Artemis 方案的对比

| | KRKR2 方案（本 skill） | Artemis 方案（artemis-*） |
|---|---|---|
| 数据 | `.xp3`，**通常不用解包** | `root.pfs`，需解包（Artroid）或原样内置（Tyranor） |
| 运行器 | Kirikiroid2（成熟、社区大） | Tyranor / 官方 Artemis |
| 单 APK | 需自己编译 KRKR2（工作量大） | 有现成壳可改 |
| 上手难度 | **低**（装 APK + 放目录） | 中（要解包/打包） |

**一句话**：KiriKiri 游戏优先用 KRKR2 直接跑；只有强需求"单 APK 点开即玩"才走定制。
