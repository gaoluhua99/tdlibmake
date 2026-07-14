# TDLib Make for Android

使用 GitHub Actions 自动编译 [TDLib (Telegram Database Library)](https://github.com/tdlib/td) 的 Android JNI 库 (`libtdjni.so`)，并生成 Java API 绑定文件。

## 产物

每次成功构建会生成 `tdlib-android-<版本号>.zip`，包含：

```
tdlib-android-<版本号>.zip
├── jniLibs/
│   ├── arm64-v8a/libtdjni.so
│   ├── armeabi-v7a/libtdjni.so
│   ├── x86_64/libtdjni.so
│   └── x86/libtdjni.so
├── java/org/drinkless/tdlib/
│   ├── TdApi.java
│   └── Client.java
└── VERSION.txt
```

- **4 个 ABI**：`arm64-v8a`、`armeabi-v7a`、`x86_64`、`x86`
- **minSdkVersion**：21（Android 5.0+）
- **OpenSSL**：3.2.1（静态链接）
- **NDK**：r26d

## 使用方法

### 方式一：推送 Tag（推荐）

适用于常规版本发布。

1. 修改 `tdlib.version` 文件，写入要编译的 TDLib 上游 commit SHA 或 tag（例如 `a17f87c4cff7b90b278d12b91ba0614383aaee82` 或 `v1.8.0`）
2. 提交并推送
3. 创建并推送版本 tag（必须以 `v` 开头）：

```bash
git tag v1.8.65
git push origin v1.8.65
```

GitHub Actions 会自动：
- 从 `tdlib.version` 读取 TDLib 上游 ref
- 编译 4 个 ABI 的 `libtdjni.so`
- 生成 `TdApi.java` 和 `Client.java`
- 打包为 `tdlib-android-v1.8.65.zip`
- 创建 GitHub Release 并上传附件

### 方式二：手动触发（Workflow Dispatch）

适用于测试或自定义构建。

1. 打开 GitHub 仓库页面 → **Actions** → **Build TDLib for Android** → **Run workflow**
2. 填写参数：

| 参数 | 说明 | 示例 |
|------|------|------|
| `tdlib_ref` | tdlib/td 上游的 commit SHA 或 tag | `a17f87c4cff7b90b278d12b91ba0614383aaee82` |
| `release_tag` | 本次 Release 的版本号（留空自动生成） | `v1.8.65` |
| `prerelease` | 是否标记为预发布 | `false` / `true` |

## 接入 Android 项目

将解压得到的文件集成到你的 Android 项目中：

1. **jniLibs**：将 `jniLibs/` 目录复制到 `app/src/main/` 下
2. **Java API**：将 `java/` 下的 `org.drinkless.tdlib` 包合并到 `app/src/main/java/` 目录

最终项目结构：

```
app/src/main/
├── jniLibs/
│   ├── arm64-v8a/libtdjni.so
│   ├── armeabi-v7a/libtdjni.so
│   ├── x86_64/libtdjni.so
│   └── x86/libtdjni.so
└── java/
    └── org/drinkless/tdlib/
        ├── TdApi.java
        └── Client.java
```

## 本地构建（可选）

如果需要在本地编译，参考以下步骤（需安装 CMake、Ninja、Android NDK r26d）：

```bash
# 1. 克隆 TDLib
git clone https://github.com/tdlib/td.git
cd td

# 2. 预生成源码
mkdir build-pregenerate && cd build-pregenerate
cmake -DCMAKE_BUILD_TYPE=Release \
      -DTD_GENERATE_SOURCE_FILES=ON \
      -DTD_ENABLE_JNI=ON \
      -GNinja ..
cmake --build . -- -j$(nproc)
cd ..

# 3. 交叉编译（以 arm64-v8a 为例）
cd example/android
mkdir build-arm64-v8a && cd build-arm64-v8a
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI=arm64-v8a \
      -DANDROID_PLATFORM=android-21 \
      -DTD_ENABLE_JNI=ON \
      -GNinja ..
cmake --build . --target tdjni -- -j$(nproc)
```

## 项目文件说明

| 文件 | 说明 |
|------|------|
| `.github/workflows/build-tdlib.yml` | GitHub Actions 工作流定义 |
| `tdlib.version` | 指定要编译的 tdlib/td 上游 ref（commit SHA 或 tag） |
