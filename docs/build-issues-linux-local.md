# Linux Local Build Issues & Warnings Analysis

> Date: 2026-05-09
> Build preset: `linux-local`
> CMake version: 4.3
> Platform: Ubuntu (x86_64)

---

## 1. 编译错误修复记录

### 1.1 Imath 库查找失败

**报错信息：**
```
CMake Error: Could not find a package configuration file provided by "Imath"
(requested version 3.1.9)
```

**原因：** `CMakeLists.txt:664` 中 `CPMAddPackage` 设置了 `FIND_PACKAGE_ARGUMENTS "REQUIRED"`，
导致 CPM 优先调用 `find_package(Imath)` 查找系统安装版本。系统未安装或已安装的 apt 版本
（`libimath-dev`）CMake 配置文件与当前 CMake 版本不兼容，`REQUIRED` 使其直接报错退出，
没有机会回退到 URL 下载路径。

**修复方案：** 注释掉 `FIND_PACKAGE_ARGUMENTS "REQUIRED"`，让 CPM 直接从 GitHub 下载源码构建。

```diff
# CMakeLists.txt:664
- FIND_PACKAGE_ARGUMENTS "REQUIRED"
+ # FIND_PACKAGE_ARGUMENTS "REQUIRED"
```

### 1.2 OpenEXR 库查找失败

**报错信息：**
```
CMake Error: Could not find a package configuration file provided by "OpenEXR"
(requested version 3.1.5)
```

**原因：** 与 Imath 完全相同的问题。`CMakeLists.txt:687` 的 `FIND_PACKAGE_ARGUMENTS "REQUIRED"`
导致 `find_package(OpenEXR)` 失败后直接 FATAL_ERROR，无法回退到 URL 下载。

**修复方案：** 同样注释掉 `FIND_PACKAGE_ARGUMENTS "REQUIRED"`。

```diff
# CMakeLists.txt:687
- FIND_PACKAGE_ARGUMENTS "REQUIRED"
+ # FIND_PACKAGE_ARGUMENTS "REQUIRED"
```

### 1.3 TIFF 库 Git Tag 失效

**报错信息：**
```
fatal: reference is not a tree: b6446af165a5a4177eb9a4f6834bb948d4f89d39
Failed to checkout tag: 'b6446af165a5a4177eb9a4f6834bb948d4f89d39'
```

**原因：** `CMakeLists.txt:1168` 使用 fork 仓库 `Tom94/libtiff`，指定的 commit hash
`b6446af...` 在远程仓库中已不存在（可能被 force push 或 rebase 覆盖）。

**修复方案：** 更新为仓库当前 HEAD 的 commit hash。

```diff
# CMakeLists.txt:1168
- GIT_TAG b6446af165a5a4177eb9a4f6834bb948d4f89d39
+ GIT_TAG 392d93518a813fe407e119aeb8596eb4891f3fd1
```

### 1.4 Imath 缓存残留

**现象：** 卸载 `libimath-dev` 并注释掉 `REQUIRED` 后，仍报错：
```
-- CPM: Using local package Imath@3.1.12
CMake Error: Imath library not found.
```

**原因：** CPM 在 `_deps/` 目录中缓存了之前不完整的 Imath 构建产物，`find_package`
找到了缓存但其中缺少 `Imath::Imath` target。

**修复方案：** 清理 CPM 缓存和 CMakeCache：

```bash
rm -rf build/linux-local/_deps/imath-build \
       build/linux-local/_deps/imath-src \
       build/linux-local/_deps/imath-subbuild \
       build/linux-local/CMakeCache.txt
```

---

## 2. 编译 Warning 分析

### 2.1 CMake 弃用警告（无影响，无需修复）

| 来源 | 文件 | 内容 | 影响 |
|------|------|------|------|
| freetype | `freetype-src/CMakeLists.txt:113` | `cmake_minimum_required` 版本 < 3.10 | 零 — 仅提示未来 CMake 可能移除旧版兼容 |
| portable-file-dialogs | `portable-file-dialogs-src/CMakeLists.txt:1` | 同上 | 零 |
| JXL/highway | `jxl-src/third_party/highway/CMakeLists.txt:28` | `CMP0111` policy 设为 OLD | 零 |

以上均为**上游依赖的 CMakeLists.txt 问题**，不影响编译产物正确性和运行时功能。
待上游更新后自动消失。

### 2.2 缺失可选依赖（影响部分格式支持）

| 缺失库 | 触发位置 | 功能影响 | 严重程度 |
|--------|----------|----------|----------|
| **LIBDE265** | libheif | HEIC 文件**无法解码**（编码正常） | 低 |
| **OpenH264** | libheif | H.264 编解码不可用 | 极低 |
| **OPENJPH** | libheif + OpenEXR | HT-JPEG2000 **只能解码，不能编码** | 低 |
| **BZip2** | freetype/plutosvg | 可选压缩格式缺失 | 零 — 不影响字体渲染 |
| **Jasper (JP2K)** | LibRaw | JPEG2000 RAW 解码回退到其他路径 | 低 |
| **ZSTD** | LibTIFF | TIFF ZSTD 压缩不支持 | 低 |
| **LERC** | LibTIFF | TIFF LERC 压缩不支持 | 低 |
| **WebP** | LibTIFF | TIFF WebP 压缩不支持 | 低 |
| **GLUT** | LibTIFF | OpenGL 工具不可用 | 零 — hdrview 不使用 |

**最终 codec 支持矩阵：**

| 格式 | 解码 | 编码 | 备注 |
|------|------|------|------|
| AVIF | YES | YES | 完整 |
| HEIC | NO | YES | 解码需安装 `libde265-dev` |
| JPEG | YES | YES | 完整 |
| JPEG2000 | YES | YES | 完整 |
| HT-JPEG2000 | YES | NO | 编码需安装 OpenJPH 开发包 |

### 2.3 运行时库路径冲突警告（建议关注）

**报错信息（重复 4 次）：**
```
Cannot generate a safe runtime search path for target HDRView because files in some
directories may conflict with libraries in implicit directories:

  runtime library [libpng16.so.16] in /usr/lib/x86_64-linux-gnu may be hidden by files in:
      /home/linuxbrew/.linuxbrew/lib
  runtime library [libjpeg.so.8] in /usr/lib/x86_64-linux-gnu may be hidden by files in:
      /home/linuxbrew/.linuxbrew/lib
  runtime library [libz.so.1] in /usr/lib/x86_64-linux-gnu may be hidden by files in:
      /home/linuxbrew/.linuxbrew/lib
  runtime library [libgomp.so.1] in /usr/lib/gcc/x86_64-linux-gnu/11 may be hidden by files in:
      /home/linuxbrew/.linuxbrew/lib
```

**原因：** 系统通过 linuxbrew 安装了 harfbuzz 等库，brew 的 lib 目录
(`/home/linuxbrew/.linuxbrew/lib`) 与系统 lib 目录 (`/usr/lib/x86_64-linux-gnu`)
存在同名 `.so` 文件。CMake 的 RPATH 生成机制无法确定应优先加载哪个版本的库。

**潜在风险：**
- 运行时可能加载 brew 版本的 libpng/libjpeg/libz，而非系统版本
- 如果 brew 版本与编译时链接的系统版本 ABI 不一致，可能导致 segfault 或符号解析错误

**建议修复方案（二选一）：**

**方案 A — 排除 linuxbrew 路径（推荐）：**
在 `CMakeLists.txt` 或 preset 中添加：
```cmake
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)
```

**方案 B — 运行时控制环境变量：**
确保启动程序时不包含 brew 路径：
```bash
unset LD_LIBRARY_PATH  # 或确保不包含 ~/.linuxbrew/lib
./HDRView
```

---

## 3. 总结

| 类别 | 数量 | 是否需要处理 |
|------|------|-------------|
| 致命编译错误 | 3（Imath/OpenEXR/TIFF） | 已修复 |
| CMake 弃用警告 | 3 | 无需处理（上游问题） |
| 可选依赖缺失 | ~9 个 | 按需安装，不装也不影响核心功能 |
| RPATH 冲突警告 | 1（重复 4 次） | **建议处理**，避免运行时崩溃 |
