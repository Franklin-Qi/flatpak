# Flatpak 技术框架结构分析

> 个人学习笔记分支，基于本仓库 `main` 源码梳理。  
> **不面向 upstream PR**；仅作远程仓库记录。

本文说明 Flatpak **本仓库**如何组织代码、进程如何协作，以及应用从「安装」到「沙箱运行」的主路径。对外产品文档仍以 [docs.flatpak.org](https://docs.flatpak.org/) 为准。

---

## 1. 一句话定位

Flatpak = **以 OSTree 为内容存储与分发** + **以 bubblewrap 做进程/文件系统沙箱** + **以 D-Bus Portal / Permission Store 做受控越权访问** 的桌面应用交付栈。

本仓库提供：

| 产物 | 作用 |
|------|------|
| `flatpak` CLI (`app/`) | 用户与发行版脚本入口 |
| `libflatpak` (`common/`) | 安装、事务、运行时上下文、导出给 gnome-software 等 |
| `flatpak-system-helper` | 系统级安装的特权操作（Polkit） |
| `flatpak-session-helper` | 每用户会话侧辅助 |
| `flatpak-portal` | `org.freedesktop.portal.Flatpak` 实现（子沙箱等） |
| `flatpak-oci-authenticator` | OCI 远程认证 |
| `revokefs` | 可撤销 FUSE 文档挂载相关 |

外部依赖（系统或 wrap）：**OSTree**、**bubblewrap (`bwrap`)**、**xdg-dbus-proxy**、**xdg-desktop-portal**（文档/权限等由门户生态提供）。

---

## 2. 总览架构图

```mermaid
flowchart TB
  subgraph UserSpace["用户空间"]
    CLI["flatpak CLI\n(app/)"]
    LIB["libflatpak\n(common/)"]
    APP["沙箱内应用"]
  end

  subgraph Priv["特权 / 会话服务"]
    SH["flatpak-system-helper\n+ Polkit"]
    SEH["flatpak-session-helper"]
    PORTAL["flatpak-portal"]
    OCI["flatpak-oci-authenticator"]
  end

  subgraph Host["宿主机能力"]
    OSTREE["OSTree repo\n/var/lib/flatpak\n~/.local/share/flatpak"]
    BWRAP["bubblewrap"]
    DBUS["xdg-dbus-proxy"]
    XDP["xdg-desktop-portal\nDocuments / PermissionStore"]
  end

  CLI --> LIB
  LIB -->|user 安装| OSTREE
  LIB -->|system 安装| SH
  SH --> OSTREE
  LIB -->|flatpak run| BWRAP
  BWRAP --> APP
  LIB --> DBUS
  APP --> PORTAL
  APP --> XDP
  LIB --> SEH
  LIB --> OCI
```

---

## 3. 源码目录地图

```mermaid
flowchart LR
  ROOT["flatpak/"]
  ROOT --> APPDIR["app/\nCLI builtins"]
  ROOT --> COMMON["common/\nlibflatpak 核心"]
  ROOT --> SYSH["system-helper/"]
  ROOT --> SESH["session-helper/"]
  ROOT --> PORT["portal/"]
  ROOT --> OCIAUTH["oci-authenticator/"]
  ROOT --> REV["revokefs/"]
  ROOT --> DATA["data/\nD-Bus XML"]
  ROOT --> DOC["doc/\nman pages"]
  ROOT --> TESTS["tests/"]
  ROOT --> TRIG["triggers/"]
  ROOT --> PROF["profile/ env.d/\nXDG_DATA_DIRS"]
  ROOT --> SUB["subprojects/\nlibglnx …"]
```

| 目录 | 功能说明 |
|------|----------|
| `app/` | `flatpak` 可执行文件：每个子命令一个 `flatpak-builtins-*.c`，由 `flatpak-main.c` 分发 |
| `common/` | 共享库：目录/远程/事务/上下文/bwrap 参数/运行时挂载 |
| `system-helper/` | 系统安装 D-Bus 服务 + Polkit policy |
| `session-helper/` | 用户会话 D-Bus 服务 |
| `portal/` | Flatpak 专用 portal（如 spawn / 子沙箱） |
| `oci-authenticator/` | OCI registry 认证 helper |
| `revokefs/` | 文档门户相关的可撤销文件系统 |
| `data/` | 对外 D-Bus 接口 XML |
| `doc/` | DocBook man pages |
| `tests/` | 集成测试 / wrap |
| `triggers/` | 安装后触发器（桌面文件、图标等） |
| `profile/` / `env.d/` | 让宿主桌面看到 exports（`XDG_DATA_DIRS`） |
| `completion/` | bash/zsh 补全 |
| `meson.build` | 构建与可选组件（system-helper、selinux、sideload 等） |

---

## 4. 核心库分层（`common/`）

```mermaid
flowchart TB
  subgraph API["对外 / 稳定倾向 API"]
    INST["FlatpakInstallation"]
    TRANS["FlatpakTransaction"]
    REF["FlatpakRef / InstalledRef / RemoteRef"]
  end

  subgraph Core["内部核心"]
    DIR["FlatpakDir\n安装根 + OSTree"]
    CTX["FlatpakContext\n权限 / filesystem / env"]
    RUN["flatpak-run*\n组装 bwrap"]
    EXP["FlatpakExports\nhost 路径导出"]
    BWRAP["FlatpakBwrap\n参数与 fd"]
  end

  subgraph Content["内容与远程"]
    REPO["repo-utils / bundle"]
    OCI["oci-registry"]
    REM["FlatpakRemote"]
  end

  INST --> DIR
  TRANS --> DIR
  DIR --> REPO
  DIR --> OCI
  RUN --> CTX
  RUN --> EXP
  RUN --> BWRAP
  TRANS --> REM
```

### 4.1 关键模块

| 模块（文件前缀） | 职责 |
|------------------|------|
| `flatpak-dir` | 一个「安装」：user/system/custom；管理 remotes、部署、override、summary |
| `flatpak-installation` | 库 API 封装 `FlatpakDir` |
| `flatpak-transaction` | install/update/uninstall 的依赖解析与执行（pull + deploy） |
| `flatpak-context` | 沙箱权限模型：`shared` / `sockets` / `devices` / `filesystems` / `persist` / env |
| `flatpak-exports` | 按 permissions 决定宿主路径如何 bind 进沙箱 |
| `flatpak-run*` | `flatpak run`：环境、X11/Wayland/Pulse/D-Bus/Cups、实例目录 |
| `flatpak-bwrap` | 构造 bubblewrap 命令行与 memfd/fd 传递 |
| `flatpak-ref-utils` | `app\|runtime/id/arch/branch` 分解与模糊匹配 |
| `flatpak-repo-utils` | summary、bundle（`.flatpak`）、签名相关 |
| `flatpak-oci-*` | OCI 镜像远程 |
| `flatpak-instance` | 运行中实例（`ps` / `kill` / `enter`） |

---

## 5. CLI 功能分组（`app/`）

```mermaid
mindmap
  root((flatpak CLI))
    生命周期
      install / update / uninstall
      mask / pin / preinstall
      repair / history / config
    查询
      list / info / search
      remotes / remote-ls / remote-info
    运行与权限
      run / override / make-current
      enter / ps / kill
    门户文档与权限库
      document-*
      permission-*
    构建与仓库
      build-init / build / build-finish
      build-export / build-bundle
      build-sign / repo / create-usb
```

入口：`app/flatpak-main.c` 将子命令映射到 `flatpak_builtin_*`。事务型命令走 `FlatpakTransaction`（CLI 包装在 `flatpak-cli-transaction.c`）。

---

## 6. 安装路径：事务与特权边界

```mermaid
sequenceDiagram
  participant U as User / CLI
  participant L as libflatpak
  participant P as Polkit
  participant H as system-helper
  participant O as OSTree repo

  U->>L: flatpak install --system …
  L->>L: FlatpakTransaction 解析依赖
  alt user 安装
    L->>O: pull + deploy (~/.local/share/flatpak)
  else system 安装
    L->>H: D-Bus Deploy / Pull …
    H->>P: 鉴权
    P-->>H: 允许 / 拒绝
    H->>O: pull + deploy (/var/lib/flatpak)
  end
  L-->>U: 完成 / 错误
```

要点：

- **User vs System**：两套（或更多 `installations.d`）独立 OSTree + 部署树。
- **system-helper**：避免日常 CLI 长期以 root 运行；写系统树经 Polkit。
- **Remote**：OSTree / OCI；Flathub 等为配置项，不在本仓库内置内容。

磁盘布局（简化）：

```text
$prefix/var/lib/flatpak/          # 默认 system
  repo/                           # OSTree
  app/<id>/<arch>/<branch>/
  runtime/...
  overrides/
  exports/                        # 导出到宿主的 desktop/icon 等

~/.local/share/flatpak/           # 默认 user（同上结构）
~/.var/app/<id>/                  # 应用可写数据 (XDG_* 映射)
```

---

## 7. 运行路径：沙箱启动

```mermaid
sequenceDiagram
  participant CLI as flatpak run
  participant Ctx as FlatpakContext
  participant Exp as Exports
  participant Bw as bwrap
  participant App as App process
  participant Proxy as xdg-dbus-proxy

  CLI->>Ctx: 合并 metadata + overrides + CLI 参数
  CLI->>Exp: 计算 filesystem 绑定
  CLI->>Bw: 组装命名空间 / 挂载 / 环境
  CLI->>Proxy: 过滤会话/系统总线（按需）
  Bw->>App: exec 应用入口
  App-->>CLI: 生命周期由实例目录跟踪
```

权限合并顺序（部署时 load overrides，简化）：

1. Runtime / App metadata 中的 `[Context]`
2. System-wide `global` → system per-app
3. Per-user `global` → user per-app
4. 命令行 `--filesystem` / `--socket` 等

`filesystem=host` 等语义见 `flatpak-metadata(5)`；实现上对根目录做 **denylist** 导出，并对 `/run/media` 等特殊处理。

---

## 8. Portal / 会话服务

```mermaid
flowchart LR
  APP["沙箱应用"] -->|"受限 D-Bus"| PROXY["xdg-dbus-proxy"]
  PROXY --> BUS["宿主 session bus"]
  APP --> FP["flatpak-portal\norg.freedesktop.portal.Flatpak"]
  APP --> XDP["xdg-desktop-portal.*\nDocuments / FileChooser / …"]
  XDP --> PS["PermissionStore"]
  FP --> SPAWN["受控 spawn / 子沙箱"]
```

| 组件 | 接口 / 角色 |
|------|-------------|
| `session-helper` | `org.freedesktop.Flatpak`（会话辅助） |
| `portal` | `org.freedesktop.portal.Flatpak` |
| 外部 xdg-desktop-portal | 文档、截图、设备等跨应用能力 |
| Permission Store | `flatpak permission-*` 与门户授权持久化 |

应用 **不应** 默认获得宿主全盘；按需经 portal 授权是安全模型的核心。

---

## 9. 构建与发行（仓库内工具）

开发者/打包侧常用子命令（仍由本 CLI 提供；高级清单构建多用外部 **flatpak-builder**）：

```mermaid
flowchart LR
  INIT["build-init"] --> BUILD["build"]
  BUILD --> FIN["build-finish"]
  FIN --> EXP["build-export"]
  EXP --> REPO["OSTree repo"]
  REPO --> BUNDLE["build-bundle\n.flatpak"]
  REPO --> USB["create-usb"]
  REPO --> SIGN["build-sign"]
```

- **Runtime / SDK / App** 分层：应用依赖 runtime；SDK 用于构建。
- **Extensions**（Locale、GL、Codecs）由 metadata 声明，事务自动拉取。

---

## 10. 测试与构建系统

| 方面 | 说明 |
|------|------|
| 构建 | Meson；可选 system-helper、selinux、sideload systemd、installed-tests |
| 测试 | `tests/*.sh` + `.wrap`；本地可用 `_build/app/flatpak` |
| 隔离 | 测试常用 `FLATPAK_USER_DIR` 等指向临时目录 |
| CI | `.github/workflows/`（gcc/asan、clang、shellcheck、potfiles 等） |

---

## 11. 与生态项目的边界

```mermaid
flowchart TB
  FP["flatpak\n本仓库"]
  FB["flatpak-builder\n清单构建"]
  FH["Flathub\n应用仓库"]
  FM["flat-manager\n仓库托管"]
  FS["Flatseal\n权限 GUI"]
  XDP2["xdg-desktop-portal"]

  FB -->|产出 refs| FH
  FH -->|remote| FP
  FM --> FH
  FS -->|写 overrides| FP
  FP --> XDP2
```

---

## 12. 阅读源码的推荐路径

1. **CLI 分发**：`app/flatpak-main.c`
2. **安装事务**：`common/flatpak-transaction.c` → `flatpak-dir.c`
3. **权限模型**：`common/flatpak-context.c` + `doc/flatpak-metadata.xml`
4. **启动沙箱**：`common/flatpak-run.c` + `flatpak-bwrap.c` + `flatpak-exports.c`
5. **系统特权**：`system-helper/flatpak-system-helper.c` + `*.policy.in`
6. **Portal**：`portal/flatpak-portal.c` + `data/org.freedesktop.portal.Flatpak.xml`

---

## 13. 修订记录

| 日期 | 说明 |
|------|------|
| 2026-10-16 | 初稿：基于当时 `main` 目录与组件关系整理 |

分支：`notes/technical-architecture`（仅推送到个人 fork，不提交 upstream PR）。
