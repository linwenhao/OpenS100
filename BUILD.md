# 构建指南（Windows / Visual Studio / vcpkg）

本文档给出一个在 Windows 下经过验证且稳定的构建方案：  
使用 Visual Studio + vcpkg（manifest 模式）。

该方案基于真实构建问题沉淀而来（尤其是 `pugixml` 相关问题），
重点规避源码压缩包解压失败与 CMake/toolchain 兼容性问题。

---

## 1. 前置要求

### 1.1 操作系统
- Windows 10 / 11（x64）

### 1.2 Visual Studio
- Visual Studio 2022
- 必需工作负载：
  - **使用 C++ 的桌面开发**
- 必需组件：
  - MSVC v143 工具集
  - Windows 10/11 SDK

---

## 2. CMake 配置（重要）

### 2.1 必需 CMake 版本
- **CMake 3.29.x**

> vcpkg 内置的更新版 CMake（例如 3.31.x）在 Windows 上执行 `vcpkg install`
> 时，可能出现源码压缩包解压失败（例如 `pugixml` 的 `.tar.gz`）。

### 2.2 安装 CMake
下载并安装 CMake 3.29：
- https://cmake.org/download/

安装后请验证：
```bat
cmake --version
```

---

## 3. vcpkg 配置

### 3.1 vcpkg 目录

本文档默认 vcpkg 位于：

```text
C:\vcpkg
```

请先更新 vcpkg：

```bat
cd C:\vcpkg
git pull
vcpkg update
```

---

## 4. 强制 vcpkg 使用系统 CMake

为避免 vcpkg 内置 CMake 引发问题，建议**强制使用系统安装的 CMake**。

### 4.1 环境变量

设置以下环境变量：

```bat
VCPKG_FORCE_SYSTEM_BINARIES=1
```

作用：

* vcpkg 不再使用其内部下载的 CMake
* 改为使用系统安装的 CMake 3.29

> 设置后请重启 Visual Studio 和所有终端。

---

## 5. 推荐 vcpkg 二进制缓存配置

强烈建议启用二进制缓存，以便：

* 缩短构建时间
* 减少重复解压和重复编译
* 提升 manifest 模式下的稳定性

### 5.1 推荐设置

设置以下环境变量：

```bat
VCPKG_BINARY_SOURCES=clear;files,C:\vcpkg\binary-cache,readwrite
```

### 5.2 说明

* `clear`：清理隐式/历史缓存源
* `files,C:\vcpkg\binary-cache,readwrite`：启用本地文件缓存目录
* 对以下场景均有效：
  * 命令行构建
  * Visual Studio 构建
  * vcpkg manifest 模式

### 5.3 注意

如果已设置 `VCPKG_BINARY_SOURCES`，
则 `VCPKG_DEFAULT_BINARY_CACHE` 不再需要（会被忽略）。

---

## 6. 构建流程

### 6.1 清理历史失败状态（可选但推荐）

如果之前依赖编译失败过，可先执行：

```bat
rmdir /s /q C:\vcpkg\buildtrees
```

（请不要删除 `binary-cache`。）

---

### 6.2 在 Visual Studio 中构建

* 打开解决方案
* 选择配置：
  * `Debug | x64` 或 `Release | x64`
* 执行 Build

vcpkg 依赖会：

* 优先从二进制缓存恢复
* 必要时使用系统 CMake 构建

---

## 7. 关键配置汇总

| 项目                          | 值                                            |
| ----------------------------- | --------------------------------------------- |
| CMake                         | **3.29.x（系统安装）**                        |
| vcpkg 模式                    | Manifest                                      |
| `VCPKG_FORCE_SYSTEM_BINARIES` | `1`                                           |
| `VCPKG_BINARY_SOURCES`        | `clear;files,C:\vcpkg\binary-cache,readwrite` |
| 二进制缓存                    | 建议开启                                      |

---

## 8. 为什么这样配置

该配置可规避：

* Windows 上 CMake 解压归档失败
* CLI 与 Visual Studio 构建行为不一致
* 常见库反复源码构建（例如 `pugixml`）

该方案已在实际构建中验证，可作为项目默认基线。

---

## 9. 离线构建指南（内网/隔离网）

如果构建机器无法访问互联网，请采用**两台机器流程**：

* **机器 A（联网）**：下载并准备所有三方依赖
* **机器 B（离线）**：导入缓存后本地构建

### 9.1 本项目使用的三方库

依赖由 `vcpkg.json` 声明：

* pugixml
* geographiclib
* polyclipping
* hdf5
* libxslt
* libxml2
* openssl
* sqlite3
* boost-geometry

### 9.2 机器 A（联网）：准备依赖包

1) 安装工具（Visual Studio 2022 + CMake 3.29.x + vcpkg），并设置：

```bat
set VCPKG_FORCE_SYSTEM_BINARIES=1
set VCPKG_BINARY_SOURCES=clear;files,C:\vcpkg\binary-cache,readwrite
```

2) 在仓库根目录预构建目标 triplet 的依赖：

```bat
cd /d <repo-root>
vcpkg install --triplet x64-windows
vcpkg install --triplet x64-windows-static
```

> 如果你的工程只使用一个 triplet，只保留对应命令即可。

3) 检查缓存目录是否产生内容：

```bat
dir C:\vcpkg\binary-cache
dir C:\vcpkg\downloads
```

4) 打包以下目录并传输到离线机器：

* `C:\vcpkg\binary-cache`（编译后的二进制包）
* `C:\vcpkg\downloads`（vcpkg 下载的源码压缩包）
* `C:\vcpkg\installed`（可选，能加速首次离线构建）
* 项目源码目录（包含 `vcpkg.json`）

建议压缩包命名：

```text
opens100-offline-deps-YYYYMMDD.zip
```

### 9.3 机器 B（离线）：导入并引用依赖

1) 将目录恢复到相同 vcpkg 根路径（示例 `C:\vcpkg`）：

* `binary-cache` -> `C:\vcpkg\binary-cache`
* `downloads` -> `C:\vcpkg\downloads`
* （可选）`installed` -> `C:\vcpkg\installed`

2) 设置环境变量（必需）：

```bat
set VCPKG_FORCE_SYSTEM_BINARIES=1
set VCPKG_BINARY_SOURCES=clear;files,C:\vcpkg\binary-cache,read
```

`read` 模式可避免在受限环境中产生写入失败。

3) 确认工程使用 vcpkg manifest 模式：

* 本仓库 `.vcxproj` 已启用 `<VcpkgEnableManifest>true</VcpkgEnableManifest>`
* 依赖列表由解决方案根目录 `vcpkg.json` 统一解析

4) 离线预检查（可选）：

```bat
cd /d <repo-root>
vcpkg install --triplet x64-windows --debug
```

若缓存准备完整，此步骤不应触发外网下载。

### 9.4 三方库在工程中的引用方式

本项目使用 **vcpkg manifest 模式**，而不是手工逐个配置 include/lib：

* `vcpkg.json` 声明依赖库
* `.vcxproj` 通过 `VcpkgEnableManifest` 启用自动集成
* Visual Studio / MSBuild 通过 vcpkg 工具链自动处理头文件与链接库

因此离线构建的核心是：
**提前准备并导入 `binary-cache` + `downloads`**。

### 9.5 离线编译方式

#### 方式 A：Visual Studio

1) 打开 `OpenS100.sln`
2) 选择 `Debug|x64` 或 `Release|x64`
3) Build Solution

#### 方式 B：MSBuild 命令行

```bat
cd /d <repo-root>
msbuild OpenS100.sln /m /p:Configuration=Release /p:Platform=x64
```

### 9.6 离线问题排查

如果构建仍尝试联网，请检查：

* `VCPKG_BINARY_SOURCES` 是否在当前终端/IDE 进程中生效
* `C:\vcpkg\downloads` 是否包含所需源码包
* 离线机器构建的 triplet 是否与联网机一致
* CMake 是否为 3.29.x 且 `VCPKG_FORCE_SYSTEM_BINARIES=1` 已设置
