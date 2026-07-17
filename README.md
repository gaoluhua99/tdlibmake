# tdlibmake

自动追踪 [tdlib/td](https://github.com/tdlib/td) 上游版本，使用 GitHub Actions 编译 Android 所需的 `libtdjni.so`，并将产物打包发布到 Releases。无需本地环境，全程自动化。

## 功能特性

- **自动追踪上游**：每天定时检查 tdlib/td 的 CMakeLists.txt，发现新版本自动触发编译
- **四架构并行编译**：`arm64-v8a` / `armeabi-v7a` / `x86_64` / `x86` 同时构建，节省时间
- **编译产物缓存**：OpenSSL 静态库和 ccache 均有缓存，二次构建大幅提速
- **构建验证**：编译后自动校验 ELF 格式、ABI 匹配、JNI 入口符号完整性
- **失败自动回滚**：任一 ABI 编译失败时自动删除已创建的 tag 和 Release，避免留下半成品
- **版本号文件名**：产物以 `tdlib-android-v1.8.66.zip` 格式命名，版本一目了然

## 产物结构

```
tdlib-android-v1.8.66.zip
├── jniLibs/
│   ├── arm64-v8a/libtdjni.so
│   ├── armeabi-v7a/libtdjni.so
│   ├── x86_64/libtdjni.so
│   └── x86/libtdjni.so
├── java/org/drinkless/tdlib/
│   ├── TdApi.java        # 自动生成的 TDLib Java API
│   └── Client.java       # TDLib Java 客户端
└── VERSION.txt           # 构建信息（版本号、SHA、NDK、时间）
```

## 构建环境

| 组件 | 版本 |
|---|---|
| Android NDK | r26d |
| OpenSSL | 3.5.1（静态链接） |
| minSdkVersion | 21 |
| 运行环境 | ubuntu-latest |

## 自动化流程

```
每天 UTC 08:00
    │
    ▼
  check：读取 tdlib/td master 的 CMakeLists.txt
    ├─ 版本未变 → 跳过，结束
    └─ 发现新版本
        ├─ 更新 tdlib.version 并 commit 到 main
        └─ 触发编译
            │
            ▼
  build (4 ABI 并行)
    ├─ 成功 → release：打包 zip，发布 GitHub Release
    └─ 失败 → rollback：删除 tag 和 Release
```

版本追踪依据 `tdlib/td` 仓库 `CMakeLists.txt` 中的 `project(TDLib VERSION x.x.x)`，当前已构建版本记录在仓库根目录的 [`tdlib.version`](./tdlib.version) 文件中。

## 手动触发编译

在 [Actions → TDLib Auto Check & Build](../../actions/workflows/tdlib-auto-build.yml) 页面点击 **Run workflow**，支持以下参数：

| 参数 | 说明 | 默认值 |
|---|---|---|
| `tdlib_ref` | tdlib/td 的 commit SHA 或 tag | 自动取最新 master |
| `release_tag` | Release 版本号，如 `v1.8.66` | 自动从 CMakeLists.txt 读取 |
| `force_build` | 强制编译（即使版本未变化） | `false` |
| `prerelease` | 标记为预发布版本 | `false` |

## 在 Android 项目中接入

从 [Releases](../../releases) 下载最新的 `tdlib-android-*.zip`，解压后：

**1. 复制 .so 文件**

将 `jniLibs/` 目录整体复制到 `app/src/main/`：

```
app/src/main/jniLibs/
├── arm64-v8a/libtdjni.so
├── armeabi-v7a/libtdjni.so
├── x86_64/libtdjni.so
└── x86/libtdjni.so
```

**2. 复制 Java 源文件**

将 `java/org/drinkless/tdlib/` 下的文件复制到你的源码目录：

```
app/src/main/java/org/drinkless/tdlib/
├── TdApi.java
└── Client.java
```

**3. 加载库**

在应用启动时加载：

```java
System.loadLibrary("tdjni");
```

之后即可使用 `TdApi` 和 `Client` 进行 TDLib 开发。更多用法参考 [tdlib 官方文档](https://core.telegram.org/tdlib)。

## 仓库设置要求

首次 fork 或使用时需确认以下设置：

**Settings → Actions → General**
- Workflow permissions → 选择 **Read and write permissions**
- 勾选 **Allow GitHub Actions to create and approve pull requests**

## 相关链接

- [tdlib/td](https://github.com/tdlib/td) — TDLib 官方仓库
- [TDLib 官方文档](https://core.telegram.org/tdlib) — API 文档
- [TGX-Android/tdlib](https://github.com/TGX-Android/tdlib) — Telegram X 的 tdlib 构建参考
