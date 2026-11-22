---
name: artemis-tyranor-port
description: 把 Artemis Engine（root.pfs / .pfs 分卷）的 PC galgame 用 Tyranor-Next 定制成"一键安装版"安卓 APK：游戏数据内置在 assets、首次启动自动解压、点开直达游戏。适用于所有同样是 Artemis Engine 的 PC 游戏（判据见"识别结构"）。
---

# Artemis Engine 游戏 → 安卓单 APK（Tyranor-Next 定制方案）

## 这是干什么的

把 PC galgame 打包成一个 **安卓 APK**，装上后**点开图标直接进游戏**：
- 引擎（Tyranor-Next）+ 游戏数据（1-2GB）全部内置在 APK 里
- 首次启动自动把游戏解压到 App 私有目录，之后每次点开直接启动
- 无需用户装模拟器、无需手动放文件、无需配置扫描目录

## 识别结构（先判断能不能用本方案）

打开游戏 PC 目录，看根目录：

| 看到这些文件 | 引擎 | 能否用本方案 |
|---|---|---|
| `root.pfs` + `root.pfs.000/.001/...` | **Artemis Engine** | ✅ 本方案 |
| 只有 `root.pfs`（单包） | **Artemis Engine** | ✅ 本方案 |
| `data.xp3` / `xxx.xp3` | KiriKiri / KrKr2 | ❌ 用 Kirikiroid2，见另一 skill |
| `Scene.pck` + `Gameexe.dat` | SiglusEngine | ❌ 用 Tyranor-Next 的 Siglus 宿主 |
| `game.rpa` / `.rpy` | Ren'Py | ❌ 用 Ren'Py 官方安卓打包 |
| `.pfs` 但不是 `root.pfs` | 少见，需确认 magic | ⚠️ 看 `pf8`/`pf6` 头再定 |

**验证方法**：读 `root.pfs` 前 3 字节，是 `pf8` 或 `pf6` 就是 Artemis Engine。

## 前提条件

- **JDK 17**（`C:\Program Files\Java\jdk-17.0.1+12` 之类）
- **Android SDK**：platform `android-37.0`、build-tools `37.0.0`、NDK `28.0.13004108`、CMake `3.22.1`
  - 用 `sdkmanager` 装：`sdkmanager --sdk_root=<SDK> "platform-tools" "platforms;android-37.0" "build-tools;37.0.0" "ndk;28.0.13004108" "cmake;3.22.1"`
- **项目路径必须纯英文**（AGP/aapt2 会直接拒绝中文路径）
- 磁盘：游戏 1-2GB + APK 构建产物 ≈ 5GB

## 详细步骤

### 1. 准备 Tyranor-Next 源码

```bash
git clone --depth 1 https://github.com/Weiss-UltimateSavior/Tyranor-Next.git Tyranor-Next
```

（注意：项目名里不要带中文目录，例如 `F:\mobile_port\Tyranor-Next`）

### 2. 把游戏数据放进 assets

```bash
mkdir -p Tyranor-Next/app/src/main/assets/bundled-game
# 把游戏根目录下的 root.pfs / root.pfs.000/... / 封面.png / savedata/ 复制进去
```

只复制**游戏数据**，不要复制 `.exe`、`.cmd` 启动器。

### 3. 关闭大文件压缩（可选，加快构建）

在 `app/build.gradle.kts` 的 `androidResources { }` 里加：

```kotlin
androidResources {
    noCompress += listOf("pfs", "000", "001", "002")
}
```

### 4. 新增「一键启动」Activity

新建 `app/src/main/java/com/tyranor/next/BundledGameInstaller.kt`（安装器）：

```kotlin
package com.tyranor.next

import android.content.Context
import android.content.res.AssetManager
import com.tyranor.next.core.game.model.ScanGame
import com.tyranor.next.core.game.scan.EngineScanner
import com.tyranor.next.core.game.storage.GameLibraryFacade
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File

object BundledGameInstaller {
    private const val ASSET_ROOT = "bundled-game"
    private const val GAME_DIR_NAME = "<你的游戏目录名，英文>"
    private const val MARKER = ".installed"

    suspend fun ensureInstalled(context: Context, onProgress: (String) -> Unit = {}): ScanGame? =
        withContext(Dispatchers.IO) {
            val gameDir = File(context.getExternalFilesDir(null) ?: context.filesDir, GAME_DIR_NAME)
            val marker = File(gameDir, MARKER)
            if (!marker.exists()) {
                onProgress("正在解压游戏数据…")
                gameDir.mkdirs()
                copyAssets(context.assets, ASSET_ROOT, gameDir)
                marker.writeText("installed")
            }
            // 扫描根与用户手动添加的路径保持一致
            GameLibraryFacade.saveRoot(context, gameDir.absolutePath)
            onProgress("正在识别游戏…")
            val games = EngineScanner.rescanLibrary(context)
            games.firstOrNull { it.uri == gameDir.absolutePath }
        }

    private fun copyAssets(am: AssetManager, assetDir: String, destDir: File) {
        val names = am.list(assetDir) ?: return
        for (name in names) {
            val assetPath = "$assetDir/$name"
            val children = am.list(assetPath)
            if (children != null && children.isNotEmpty()) {
                copyAssets(am, assetPath, File(destDir, name))
            } else {
                File(destDir, name).also { it.parentFile?.mkdirs() }.outputStream().use { out ->
                    am.open(assetPath).use { it.copyTo(out) }
                }
            }
        }
    }
}
```

新建 `app/src/main/java/com/tyranor/next/OneClickLaunchActivity.kt`（启动入口）：

```kotlin
package com.tyranor.next

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Text
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.lifecycle.lifecycleScope
import com.tyranor.next.core.game.launch.EngineLauncher
import com.tyranor.next.theme.TyranorNextTheme
import com.tyranor.next.ui.common.ProvideAppLocale
import com.tyranor.next.ui.common.userMessage
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

class OneClickLaunchActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        var status by mutableStateOf("正在准备游戏…")
        setContent {
            TyranorNextTheme { ProvideAppLocale { Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) { Text(status) } } }
        }
        lifecycleScope.launch {
            val game = BundledGameInstaller.ensureInstalled(applicationContext) { status = it }
            if (game == null) { status = "内置游戏未识别"; return@launch }
            val result = withContext(Dispatchers.IO) {
                EngineLauncher.launch(this@OneClickLaunchActivity, game, null)
            }
            val error = result.userMessage(this@OneClickLaunchActivity)
            if (error == null) finish() else status = error
        }
    }
}
```

### 5. 改 AndroidManifest

- 声明 `OneClickLaunchActivity`（`android:exported="true"`）
- 把两个 launcher `activity-alias`（`LauncherDefault` / `LauncherClassic`）的 `android:targetActivity` 从 `.MainActivity` 改成 `.OneClickLaunchActivity`

（MainActivity 保留但不挂 MAIN/LAUNCHER，仍可被设置页/快捷方式调用）

### 6. 改应用名 / 图标

- `app/src/main/res/values/strings.xml` 的 `app_name` 改成游戏名
- 图标：`drawable-nodpi/ic_launcher_art.png`（adaptive icon 背景层）+ `mipmap-*/ic_launcher*.png`，用游戏封面裁剪成正方形生成（脚本略）
- 原图标先备份

### 7. 构建

```bash
set JAVA_HOME=C:\Program Files\Java\jdk-17.0.1+12
set ANDROID_HOME=<SDK 路径>
gradlew.bat assembleRelease --no-daemon
# 产物：app/build/outputs/apk/release/app-release.apk
```

release 用 debug 签名（未配 CI 签名时项目默认），可直接安装。

### 8. 验证

- 卸载旧版 → 安装 → 从桌面图标启动
- 预期：显示设置进度 → 自动进标题（首次约 30 秒，之后每次 3-5 秒）
- 验证：全 CG 解锁（画廊）、进剧情、视频播放

## 关键坑

1. **中文路径**：AGP 报 `Your project path contains non-ASCII characters` —— 项目必须放纯英文路径（`F:\mobile_port\...`），不是加 `overridePathCheck=true` 就行（aapt2 仍可能失败）。
2. **1.9GB assets 构建慢**：首次构建 10 分钟级（含下载 Gradle/依赖），设 `-Xmx6g`。
3. **首次解压 UI**：解压 1.9GB 需要 1-3 分钟，要给用户进度反馈（文字/进度条），否则用户以为卡死。
4. **存储权限**：App 用 `getExternalFilesDir`（私有外部目录），Android 11+ 不需要额外权限；不要解压到 `/sdcard/Tyranor`（需要 MANAGE_EXTERNAL_STORAGE 授权，真机麻烦）。
5. **存档迁移**：PC 的 `savedata/`（`system.dat` 全 CG 解锁）可以整体复制到解压后的游戏目录，Tyranor 的 Artemis 宿主兼容 PC 存档（实测《虏之旋律》全 CG 解锁成功）。
6. **动态视频**：Tyranor 的 Artemis 视频用 IJK 播放器，支持；但首次播放若崩溃，检查模拟器资源（LDPlayer 曾因此崩过一次）。

## 可适配范围

- **所有 Artemis Engine 游戏**（`root.pfs` / `.pfs` 分卷）：换游戏只需改 `GAME_DIR_NAME` + 替换 assets 内容
- 分卷数不限（`root.pfs.000/.001/.002/...` 都复制进 assets）
- 单包 `root.pfs` 也可
- **不适用**：KiriKiri（.xp3）、Siglus、Ren'Py、Unity 等其他引擎

## 交付产物

- `app-release.apk`（游戏 + 引擎，1-2GB）
- 源码（可复用的定制工程）
- 首次启动说明（解压时间、存档位置）
