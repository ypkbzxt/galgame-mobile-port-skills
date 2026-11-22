# 第三方来源与归属

本仓库的文档和参考代码在编写过程中参考了以下第三方项目/资料。这里列出它们的来源与许可，便于合规与追溯。

## 项目

### Tyranor-Next
- 地址：https://github.com/Weiss-UltimateSavior/Tyranor-Next
- 用途：`artemis-tyranor-port` 技能的宿主工程与调用接口（游戏扫描、启动、存档）。
- 许可：以项目仓库声明为准（使用前请阅读其 LICENSE）。
- 说明：本仓库只引用其公开 API 的调用方式，不复制其源码；构建时由使用者自行克隆。

### apktool
- 地址：https://github.com/iBotPeaches/Apktool
- 用途：`artemis-artroid-port` 技能中的 APK 反编译与回编译。
- 许可：Apache License 2.0。

### Artemis Android 运行时（com.ies_net.artemis）
- 用途：作为「壳」的参考，说明官方 Artemis 引擎在 Android 上的启动方式（`android.app.lib_name=artemis`）。
- 本仓库**不分发**该运行时的二进制文件。使用者需要从自己拥有的、同引擎的官方/授权安卓版本中获取对应组件。

### PFS 格式技术文档
- 参考：https://sakari.top/posts/2025/artemis-pfs/
- 用途：`pf8` / `pf6` 归档格式的解析说明（header、文件表、SHA-1 密钥、XOR 加密）。
- 本仓库的 Python 解包脚本为按该格式说明独立实现。

## 免责声明

本仓库不包含任何游戏版权内容（游戏数据、图片、音频、脚本、存档）。

技能仅用于把**使用者合法拥有**的游戏文件打包到**使用者本人拥有的设备**上。不得用于：
- 分发游戏或修改后的游戏；
- 绕过任何 DRM、授权或正版验证；
- 任何商业用途。

如你发现本仓库有未正确署名或侵犯权利的内容，请提交 Issue，我们会尽快处理。
