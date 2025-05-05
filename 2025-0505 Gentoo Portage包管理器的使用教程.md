
## ✅ **Gentoo Portage 使用指南**

### **目录**
1. [安装软件包](#1-安装软件包)  
2. [搜索软件包](#2-搜索软件包)  
3. [更新系统](#3-更新系统)  
4. [卸载软件包](#4-卸载软件包)  
5. [查看已安装的软件包](#5-查看已安装的软件包)  
6. [清理编译缓存](#6-清理编译缓存)  
7. [配置 USE 标志](#7-配置-use-标志)  
8. [处理依赖关系](#8-处理依赖关系)  
9. [从源码构建软件包](#9-从源码构建软件包)  
10. [常见问题与解决方案](#10-常见问题与解决方案)  
11. [Portage 高级配置](#11-portage-高级配置)  
12. [预编译包与二进制分发](#12-预编译包与二进制分发)  
13. [Portage 工具链优化](#13-portage-工具链优化)  
14. [Portage 配置文件详解](#14-portage-配置文件详解)

---

### **1. 安装软件包**

#### **基本语法**
```bash
sudo emerge --ask <package_name>
```

#### **示例**
```bash
# 安装 Vim 编辑器
sudo emerge --ask app-editors/vim

# 同时安装多个包
sudo emerge --ask app-editors/vim net-misc/curl dev-vcs/git
```

#### **常用参数**
- `--ask`：交互确认操作。
- `--verbose`：显示详细输出（默认已启用）。
- `--update`：升级包到最新版本。
- `--deep`：递归更新依赖项。
- `--newuse`：根据 USE 标志变化重新编译。

---

### **2. 搜索软件包**

#### **使用 `emerge` 搜索**
```bash
emerge --search <keyword>
```
示例：
```bash
emerge --search python
```

#### **使用 `eix`（推荐）**
安装 `eix`：
```bash
sudo emerge --ask app-portage/eix
```

搜索软件包：
```bash
eix <keyword>
```
示例：
```bash
eix firefox
```

#### **搜索技巧**
- `eix -I`：仅显示已安装的包。
- `eix -U`：更新 eix 数据库（需定期运行）。

---

### **3. 更新系统**

#### **同步 Portage 树**
```bash
sudo emerge --sync
```

#### **升级世界集合（World Set）**
```bash
sudo emerge --ask --update --deep --newuse @world
```

#### **升级系统集合（System Set）**
```bash
sudo emerge --ask --update --deep --newuse @system
```

#### **清理旧依赖**
```bash
sudo emerge --ask --depclean
```

---

### **4. 卸载软件包**

#### **基本卸载**
```bash
sudo emerge --ask --unmerge <package_name>
```
示例：
```bash
sudo emerge --ask --unmerge app-editors/vim
```

#### **清理无用依赖**
```bash
sudo emerge --ask --depclean
```

#### **安全卸载（推荐）**
```bash
sudo emerge --ask --unmerge --deselect <package_name>
```
自动从 `world` 文件中移除包。

---

### **5. 查看已安装的软件包**

#### **列出所有已安装包**
```bash
qlist -I
```
需安装 `app-portage/portage-utils`：
```bash
sudo emerge --ask app-portage/portage-utils
```

#### **查看特定包信息**
```bash
equery list <package_name>
```
需安装 `app-portage/gentoolkit`：
```bash
sudo emerge --ask app-portage/gentoolkit
```

#### **查看包依赖树**
```bash
equery depends <package_name>
```

---

### **6. 清理编译缓存**

#### **删除未使用的源码**
```bash
sudo eclean-dist
```
需安装 `app-portage/gentoolkit`。

#### **删除临时编译目录**
```bash
sudo rm -rf /var/tmp/portage/*
```

#### **清理旧版本的二进制包**
```bash
sudo eclean-pkg -d -n <days>
```
示例：删除 7 天前的二进制包：
```bash
sudo eclean-pkg -d -n 7
```

---

### **7. 配置 USE 标志**

#### **全局 USE 标志**
编辑 `/etc/portage/make.conf`：
```bash
USE="X gtk -qt5"
```

#### **针对特定包的 USE 标志**
创建 `/etc/portage/package.use` 文件：
```bash
# 启用 Vim 的 Python 支持
app-editors/vim X python
```

#### **使用目录结构管理 USE 标志**
Portage 支持 `/etc/portage/package.use/` 目录，便于分类管理：
```bash
sudo mkdir -p /etc/portage/package.use/
echo "app-editors/vim X python" | sudo tee /etc/portage/package.use/vim
```

#### **查看包支持的 USE 标志**
```bash
equery uses <package_name>
```

---

### **8. 处理依赖关系**

#### **检查依赖完整性**
```bash
sudo revdep-rebuild
```
需安装 `app-portage/gentoolkit`。

#### **查看包的依赖树**
```bash
equery depends <package_name>
```

#### **强制重新编译依赖**
```bash
sudo emerge --ask --reinstall --deep <package_name>
```

#### **解决依赖冲突**
- **方法一：手动调整 USE 标志**  
  编辑 `/etc/portage/package.use`，禁用冲突的 USE 标志。
  
- **方法二：使用 `--backtrack` 参数**  
  ```bash
  sudo emerge --ask --backtrack=30 <package_name>
  ```

---

### **9. 从源码构建软件包**

#### **修改全局编译选项**
编辑 `/etc/portage/make.conf`，调整以下参数：
```bash
# CPU 架构优化
CFLAGS="-march=native -O2 -pipe"
CXXFLAGS="${CFLAGS}"

# 并行编译（CPU 核心数 + 1）
MAKEOPTS="-j5"

# 语言支持（如中文）
LINGUAS="zh_CN en_US"

# 输入设备和显卡驱动
INPUT_DEVICES="libinput"
VIDEO_CARDS="intel i965"
```

#### **强制重新编译**
```bash
sudo emerge --ask --oneshot --rebuild <package_name>
```

#### **自定义 ebuild 文件**
- **创建 overlay**  
  创建自定义仓库目录：
  ```bash
  sudo mkdir -p /usr/local/portage
  sudo chown -R portage:portage /usr/local/portage
  ```

- **添加 overlay 到 make.conf**  
  ```bash
  echo 'PORTDIR_OVERLAY="/usr/local/portage"' | sudo tee -a /etc/portage/make.conf
  ```

- **使用 `layman` 管理第三方仓库**  
  ```bash
  sudo emerge --ask app-portage/layman
  sudo layman -a <overlay_name>
  ```

---

### **10. 常见问题与解决方案**

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `No space left on device` | `/var/tmp` 分区空间不足 | 扩展 `/var/tmp` 或挂载为 tmpfs |
| `masked by: package.mask` | 包被屏蔽（如测试版） | 编辑 `/etc/portage/package.accept_keywords` |
| `USE changes` 提示 | USE 标志变更 | 运行 `emerge --ask --update --deep --newuse @world` |
| `Failed to fetch` | 网络问题或镜像不可用 | 修改 `/etc/portage/repos.conf/gentoo.conf` 中的 `sync-uri` |
| `Circular dependencies` | 循环依赖 | 启用 `mrustc-bootstrap`（如 Rust 自举问题） |

#### **修复破损的包**
```bash
sudo emerge --ask --reinstall <package_name>
```

#### **回滚到旧版本**
```bash
# 屏蔽新版本
echo ">=app-editors/vim-9.0" | sudo tee -a /etc/portage/package.mask

# 安装旧版本
sudo emerge --ask "<app-editors/vim-9.0"
```

---

### **11. Portage 高级配置**

#### **设置并行编译**
编辑 `/etc/portage/make.conf`：
```bash
# 使用 5 个线程（CPU 核心数 + 1）
MAKEOPTS="-j5"
```

#### **启用并行提取**
```bash
EMERGE_DEFAULT_OPTS="--jobs=5 --load-average=5"
```

#### **限制磁盘 I/O 影响**
```bash
# 在 make.conf 中设置
PORTAGE_NICENESS="19"
PORTAGE_IONICE="3"
```

#### **启用日志记录**
```bash
# 记录所有 emerge 操作
FEATURES="log"
PORT_LOGDIR="/var/log/portage"
```

#### **启用预编译包支持**
```bash
# 启用二进制包支持
FEATURES="binpkg-multi-instance"
```

---

### **12. 预编译包与二进制分发**

#### **启用二进制包**
编辑 `/etc/portage/make.conf`：
```bash
FEATURES="getbinpkg binpkg-multi-instance"
```

#### **生成预编译包**
```bash
sudo emerge --ask --buildpkg <package_name>
```

#### **从远程仓库安装预编译包**
```bash
sudo emerge --ask --getbinpkg <package_name>
```

#### **生成二进制包仓库**
```bash
sudo quickpkg <package_name>
```

---

### **13. Portage 工具链优化**

#### **启用 CCache 加速编译**
```bash
sudo emerge --ask dev-util/ccache
```

编辑 `/etc/portage/make.conf`：
```bash
CCACHE_SIZE="2G"
FEATURES="ccache"
```

#### **使用 Distcc 分布式编译**
```bash
sudo emerge --ask sys-devel/distcc
```

配置 `DISTCC_HOSTS` 和 `MAKEOPTS`：
```bash
DISTCC_HOSTS="localhost 192.168.1.2"
MAKEOPTS="-j10 --load-average=5"
```

#### **启用并行编译**
```bash
MAKEOPTS="-j$(nproc)"
```

---

### **14. Portage 配置文件详解**

#### **make.conf**
- **核心配置文件**：`/etc/portage/make.conf`  
  常用参数：
  ```bash
  # CPU 架构和优化
  CFLAGS="-march=native -O2 -pipe"
  CXXFLAGS="${CFLAGS}"
  
  # 并行编译
  MAKEOPTS="-j5"
  
  # 硬件驱动支持
  INPUT_DEVICES="libinput"
  VIDEO_CARDS="intel i965"
  
  # 日志和缓存
  PORT_LOGDIR="/var/log/portage"
  FEATURES="ccache log"
  ```

#### **package.use**
- **用途**：覆盖全局 USE 标志。
- **推荐目录结构**：
  ```bash
  /etc/portage/package.use/
  ├── dev-libs
  └── app-editors
  ```

#### **package.mask**
- **用途**：阻止特定包升级。
- **示例**：阻止升级到 Python 3.11
  ```bash
  >=dev-lang/python-3.11
  ```

#### **package.keywords**
- **用途**：启用测试版包（如 `~amd64` 标志）。
- **示例**：启用测试版内核
  ```bash
  ~sys-kernel/gentoo-sources-6.8.*
  ```

---

### **附录：Portage 常用命令速查表**

| 操作 | 命令 |
|------|------|
| 安装包 | `sudo emerge --ask <package>` |
| 更新系统 | `sudo emerge --ask --update --deep --newuse @world` |
| 卸载包 | `sudo emerge --ask --unmerge <package>` |
| 搜索包 | `eix <keyword>` |
| 查看包信息 | `equery list <package>` |
| 清理依赖 | `sudo emerge --ask --depclean` |
| 修复依赖 | `sudo revdep-rebuild` |
| 查看 USE 标志 | `equery uses <package>` |
| 强制重新编译 | `sudo emerge --ask --rebuild <package>` |
| 启用测试版包 | `echo "~<package>" > /etc/portage/package.keywords/<package>` |

---

### **总结**
本指南覆盖了 Gentoo Portage 的核心功能和进阶技巧，帮助您高效管理软件包。通过合理配置 USE 标志、优化编译参数、使用预编译包和工具链加速，您可以构建一个高度定制化的 Gentoo 系统。
如需进一步学习，可参考 [Gentoo Wiki - Portage](https://wiki.gentoo.org/wiki/Portage) 和 [Portage 手册](https://devmanual.gentoo.org/)。
