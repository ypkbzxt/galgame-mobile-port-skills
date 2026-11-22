---
name: artemis-artroid-port
description: 用官方 Artemis Android 运行时（com.ies_net.artemis 壳）+ 解包后的游戏数据，把 Artemis Engine（root.pfs / .pfs 分卷，pf8/pf6）PC galgame 打包成"点开即玩"的安卓单 APK。启动快（0.5 秒级）、体积小、纯 native。适用于所有 Artemis Engine 游戏。
---

# Artemis Engine 游戏 → 安卓单 APK（Artroid 式方案）

## 这是干什么的

把 PC galgame 做成**安卓单 APK**，点开**0.5 秒进标题**：
- 用**官方 Artemis Android 运行时**（`libartemis.so` + `com.ies_net.artemis` 壳）
- 游戏数据**解包**成目录（`image/ sound/ script/ system/ system.ini`）放进 assets
- 纯 native（无 Kotlin 层），比模拟器方案更快更小

## 识别结构（先判断能不能用本方案）

看游戏根目录：

| 文件 | 引擎 | 说明 |
|---|---|---|
| `root.pfs` + `.000/.001/.002` 分卷 | **Artemis Engine** | ✅ 本方案 |
| 单独 `root.pfs` | **Artemis Engine** | ✅ 本方案 |
| `data.xp3` | KiriKiri | ❌ 用 Kirikiroid2 |
| `Scene.pck` | Siglus | ❌ 用 Siglus 运行时 |

验证：读 `root.pfs` 前 3 字节 = `pf8` 或 `pf6`。

## 前提条件

- **JDK 17** + **Android SDK build-tools**（`apksigner`）
- **apktool 3.x**（`java -jar apktool.jar`）
- **Python 3 + numpy**（解包 pf8 用）
- 一个**同引擎的官方/同厂安卓移植版 APK**作为「壳」（提供 `libartemis.so` + `classes.dex` + `res/`）
  - 判断壳是否可用：`aapt2 dump xmltree <apk> --file AndroidManifest.xml` 看到 `com.ies_net.artemis.ArtemisActivity` + `android.app.lib_name="artemis"`
- 项目路径**纯英文**（aapt2 拒绝中文路径）

## 详细步骤

### 1. 解包 pf8 / pf6

Artemis 的 PFS 格式（参考 sakari.top 的技术文档）：

**Header**（小端）：
- `0x00` 3 字节 magic `pf8` / `pf6`
- `0x03` 4 字节 index_size
- `0x07` 4 字节 file_count
- `0x0B` 起：File Entries

**File Entry**（相对条目起点）：
- `0x00` 4 字节 name_len
- `0x04` name_len 字节 文件名（Windows 反斜杠分隔）
- `0x04+name_len` 4 字节 separator（0）
- `0x08+name_len` 4 字节 data_offset（绝对偏移）
- `0x0C+name_len` 4 字节 file_size

**加密（只对 pf8）**：
- key = **SHA1(index 数据)**（`archive[0x07 : 0x07+index_size]`）
- 每个文件**独立** XOR：`P_i = C_i ^ key[i % 20]`
- **pf6 不加密**（直接复制）
- **视频文件（.mp4/.wmv/.avi/.mgv 等）不加密**（即使 pf8 也是明文，XOR 会破坏它）——这是踩过的坑

**分卷**：`root.pfs.000/.001/...` 是**各自独立的 pfs**（每个有自己的 header + 文件表），不是数据分卷。逐个解包到同一目录，后解包的覆盖先解包的（补丁卷）。

核心 Python：

```python
import struct, hashlib, os
import numpy as np

VIDEO_EXTS = {'.mp4', '.wmv', '.avi', '.mgv', '.mpg', '.mpeg', '.webm', '.mkv'}

def parse_pfs(path):
    with open(path, 'rb') as f:
        data = f.read()
    version = data[:3]
    index_size = struct.unpack('<I', data[3:7])[0]
    file_count = struct.unpack('<I', data[7:11])[0]
    key = hashlib.sha1(data[7:7 + index_size]).digest()
    entries, pos = [], 0x0B
    for _ in range(file_count):
        nlen = struct.unpack('<I', data[pos:pos+4])[0]
        name = data[pos+4:pos+4+nlen].decode('utf-8', errors='replace')
        off = struct.unpack('<I', data[pos+8+nlen:pos+12+nlen])[0]
        size = struct.unpack('<I', data[pos+12+nlen:pos+16+nlen])[0]
        entries.append((name, off, size))
        pos += 16 + nlen
    return data, version, key, entries

def decrypt(data, key, off, size, version, name):
    raw = data[off:off+size]
    if version == b'pf6':
        return raw
    if os.path.splitext(name)[1].lower() in VIDEO_EXTS:
        return raw
    arr = np.frombuffer(raw, dtype=np.uint8).copy()
    ks = np.tile(np.frombuffer(key, dtype=np.uint8), (size // 20) + 1)[:size]
    return (arr ^ ks).tobytes()

def extract(path, out_dir):
    data, version, key, entries = parse_pfs(path)
    for name, off, size in entries:
        rel = name.replace('\\', '/').lstrip('/')
        target = os.path.join(out_dir, rel)
        os.makedirs(os.path.dirname(target), exist_ok=True)
        with open(target, 'wb') as f:
            f.write(decrypt(data, key, off, size, version, name))
```

对每个分卷调用 `extract`（注意顺序：主包 → `.000` → `.001` → ...，补丁卷最后）。

验证解包：
- `system.ini` 是文本（`[WINDOWS]` / `[ANDROID]` 段）
- `.png` 头 = `89 50 4E 47`
- `.otf` 头 = `OTTO`
- `movie/*.mp4` 头 = `00 00 00 xx 66 74 79 70`

### 2. 准备壳 APK

取一个**同引擎**（Artemis）的官方安卓移植版 APK 作为壳：
- 它的 `lib/arm64-v8a/libartemis.so` 就是运行时
- `classes.dex` 含 `com.ies_net.artemis.ArtemisActivity` / `VideoViewActivity`
- 包名通常是 `com.f.game` 之类

**必须先反编译看 AndroidManifest**，确认 `android.app.lib_name="artemis"`。

### 3. apktool 反编译

```bash
java -Xmx4g -jar apktool.jar d shell.apk -o work -f
```

（1.5GB APK 反编译约 5-10 分钟）

### 4. 改包名（关键，避免覆盖用户已装的同款）

**Artemis 壳的包名只出现在 AndroidManifest**（smali 里没有 `com/f/game` 引用，实测 0 个）：

改 `work/AndroidManifest.xml`：
```xml
package="com.f.game"  →  package="你的新包名"
```

（如果 smali 里确实有包名引用，需要批量替换 `com/f/game` → 新包名）

改 `work/res/values/strings.xml` 的 `app_name` 为游戏名。

### 5. 替换 assets

```bash
# 保留参考数据（改名，不删）
mv work/assets work/assets_ref
mkdir work/assets
# 复制解包后的游戏数据
cp -r pfs-extracted/* work/assets/
```

结构必须是：`system.ini + system/ + image/ + sound/ + script/ + movie/ + font/ + pa/`。

### 6. 回编译 + 签名

```bash
java -Xmx6g -jar apktool.jar b work -o unsigned.apk
apksigner sign --ks ~/.android/debug.keystore --ks-pass pass:android \
  --key-pass pass:android --out signed.apk unsigned.apk
```

（1.4GB APK 回编译约 5-10 分钟，签名 1-2 分钟）

### 7. 验证

```bash
adb install -r signed.apk
adb shell am start -n <新包名>/com.ies_net.artemis.ArtemisActivity
```

预期：0.5 秒级进标题，日志有 `libartemis.so` 加载 + `Init screen` + `CSoundDevice Oboe`。

## 关键坑

1. **pf8 加密不覆盖所有文件**：视频是明文（XOR 会破坏），必须按扩展名跳过。
2. **pf6 不加密**：直接复制。
3. **分卷是独立包**：每个 `.000/.001` 有自己的 header，不是数据分卷。
4. **中文路径**：aapt2 `failed to open directory` —— 工作目录必须纯英文。
5. **存档兼容**：
   - `system.dat`（CG 解锁）跨平台可迁（不崩溃）
   - `saveg.dat`（全局存档）PC 版迁到 Android 会 **SIGSEGV** —— 不要迁，让游戏生成默认再研究
   - Android 版存档位置：`/sdcard/Android/data/<包名>/files/`
6. **包名必须改**：否则会覆盖用户已装的同款游戏（签名冲突或直接覆盖）。
7. **assets 解包后体积**：约等于原 pfs 的 70-80%（补丁卷覆盖去重），APK 比 pfs 方案小 20-30%。

## 可适配范围

- **所有 Artemis Engine 游戏**（pf8 / pf6，单包或分卷）
- 换游戏只需：重新解包 + 替换 assets（壳和包名方案不变）
- 同一壳可复用于**所有 Artemis 游戏**（libartemis.so 与具体游戏数据无关，实测跨游戏可用）
- 同类「封包引擎 + Android 官方运行时」思路可套用：
  - **KiriKiri（.xp3）** → Kirikiroid2（KRKR2）壳，`.xp3` 一般不需要解包
  - **SiglusEngine（.pck）** → Tyranor-Next 的 Siglus 宿主
  - **BGI/Ethornell** → Tyranor-Next 的 Ethornell 宿主（也有独立壳）

## 与 Tyranor 方案的取舍

| | 本方案（Artroid） | Tyranor 定制方案 |
|---|---|---|
| 启动 | **0.5 秒** | 30 秒（首启解压） |
| APK 大小 | **1422MB** | 1965MB |
| 架构 | **纯 native** | Kotlin + 引擎 |
| 全 CG | 需 Android 存档 | PC savedata 可迁（已验证） |

**结论**：要快/小 = 本方案；要全 CG 一步到位 = Tyranor 方案。
