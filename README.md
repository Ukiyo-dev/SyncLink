# SyncLink

[English](./README.en-US.md) | 中文文档

**受 Scoop 的 `persist` 和 `shortcut` 功能启发，`synclink` 帮助你通过将配置文件或任何文件/文件夹移动到集中的同步目录（如 Dropbox、Google Drive、OneDrive 等），并在原始位置创建符号链接（symlinks）来管理它们。**

这对于将配置数据存储在非标准位置（例如 `%APPDATA%\Local`）的应用程序特别有用，这些位置可能不在 Scoop 内置持久化机制的覆盖范围内。`synclink` 自动化了移动这些文件并链接回来的过程，便于跨多台机器进行轻松备份和同步。

## 问题示例

像 [uv](https://github.com/astral-sh/uv) 这样的工具可能通过 Scoop 安装，但将配置存储在 `%APPDATA%/Local/uv` 等位置。标准的 Scoop 持久化不管理这些外部位置。`synclink` 允许你将 `uv` 配置文件夹移动到同步目录并创建符号链接，确保你的设置在无需手动干预的情况下得到备份和同步。

## 特性

*   **移动与链接：** 将目标文件或文件夹移动到指定的同步目录，并在原始路径创建符号链接。
*   **集中管理：** 跟踪所有创建的链接。
*   **快捷方式创建：** 可选地在开始菜单创建快捷方式，支持自定义名称和描述。
*   **自动提取文件描述：** 创建快捷方式时自动从 `.exe` 文件读取文件描述信息。
*   **健壮的文件处理：** 包括跨磁盘移动操作的进度指示器（已计划/已实现）。处理文件和文件夹。文件存储在同步路径内的专用 `files` 子目录中。
*   **链接维护：** 提供列出、删除（`unlink`）和重新创建（`relink`）托管链接和快捷方式的命令。
*   **配置管理：** 通过类似 `git config` 的 `config` 命令管理设置，如默认同步路径。
*   **版本迁移：** 更新到新版本时自动进行配置迁移。

## 安装

*   **下载：** 从 [Releases](https://github.com/UkiyoDevs/synclink/releases) 页面获取最新的可执行文件。
*   **Scoop（推荐）：**
    ```powershell
    scoop bucket add scoop-tools https://github.com/xuwenbolan/scoop-tools-bucket
    scoop install synclink
    ```
*   **从源代码构建：** 
    ```bash
    go mod download
    go build -o synclink
    ```

## 使用

`synclink` 通过几个命令操作：

---

### `synclink link <target_path>`

将目标文件或文件夹移动到同步目录，并在原始位置创建符号链接。

```bash
synclink link <target_path> [-n <link_name>] [-s <sync_path>] [--shortcut] [-d <description>] [--shortcut-name <name>] [--unlink]
```

**参数与选项：**

*   `<target_path>`：（必需）要管理的文件或文件夹的路径。成功执行后，此路径将成为指向同步目录中项目的符号链接。
*   `-n, --name <link_name>`：（可选）用于在 `synclink` 内标识此链接的名称。默认为 `<target_path>` 的基本名称。
*   `-s, --sync-path <sync_path>`：（可选）指定主同步根目录*内*的父目录，此特定项目应存储在其中。默认为 `DefaultSyncPath`（同步目录的根）。文件夹直接存储在此路径下，而文件存储在 `files` 子文件夹中（例如 `{sync_path}\files\{link_name}`）。
*   `--shortcut`：（可选）如果存在，在 Windows 开始菜单中为*原始* `<target_path>` 创建快捷方式。
*   `-d, --description <text>`：（可选）快捷方式的自定义描述。如果未指定，会自动从目标文件的版本信息中提取（仅适用于 `.exe` 文件）。
*   `--shortcut-name <name>`：（可选）快捷方式文件的显示名称（不同于配置名称）。默认为 `link_name`。

**示例：**

```bash
# 移动 C:\Users\You\AppData\Local\uv 并链接它，存储在默认同步路径中
synclink link C:\Users\You\AppData\Local\uv

# 移动 C:\MyTool\config.json，将链接命名为 'mytool-config'，存储在 'configs' 子同步目录
synclink link C:\MyTool\config.json -n mytool-config -s %USERPROFILE%\OneDrive\SyncedConfigs

# 创建带自定义名称的快捷方式（如果未提供描述，会自动提取）
synclink link "C:\Program Files\VSCode\Code.exe" --shortcut -n vscode --shortcut-name "Visual Studio Code"

# 创建带自定义描述和中文名称的快捷方式
synclink link "D:\Games\game.exe" --shortcut -n game -d "我最喜欢的游戏" --shortcut-name "游戏启动器"

# 仅为可执行文件创建开始菜单快捷方式，不移动或链接它
synclink link C:\ProgramFiles\MyApp\App.exe --shortcut --unlink -n MyAppLauncher
```

---

### `synclink unlink <link_name>`

删除托管的链接。

```bash
synclink unlink <link_name>
```

**参数：**

*   `<link_name>`：要删除的链接的名称（在 `link` 期间用 `-n` 指定的名称，或默认名称）。
    *   如果链接是**符号链接**：同步目录中的文件/文件夹将移回原始位置，并删除符号链接。
    *   如果链接仅作为**快捷方式**创建（使用 `--shortcut --unlink`）：仅删除开始菜单快捷方式。
    *   如果 `<link_name>` 是 `*`：尝试取消链接*所有*托管项目。谨慎使用。

**示例：**

```bash
# 取消链接名为 'uv' 的项目（移回文件，删除符号链接）
synclink unlink uv

# 删除名为 'MyAppLauncher' 的快捷方式
synclink unlink MyAppLauncher

# 取消链接所有托管项目
synclink unlink *
```

---

### `synclink relink <link_name>`

检查并在缺失或损坏时可能重新创建托管的链接或快捷方式。

```bash
synclink relink <link_name>
```

**参数：**

*   `<link_name>`：要检查的链接的名称。
    *   对于**符号链接**：验证符号链接是否存在于原始路径并正确指向。如果没有，它会尝试重新创建符号链接（假设目标仍存在于同步目录中）。它*不会*移回文件。
    *   对于**快捷方式**：验证快捷方式是否存在于开始菜单中。如果没有，它会尝试重新创建它（使用存储的描述和显示名称）。
    *   如果 `<link_name>` 是 `*`：检查并可能重新创建*所有*托管项目。

**示例：**

```bash
# 检查并可能重新创建 'uv' 的链接
synclink relink uv

# 检查并可能重新创建所有链接/快捷方式
synclink relink *
```

---

### `synclink list`

显示当前由 `synclink` 管理的所有项目的列表。

```bash
synclink list
```

**输出：** 显示 `link_name`、原始路径、同步路径目标和类型（符号链接/快捷方式）。

---

### `synclink config <get|set> <attribute> [new_value]`

获取或设置 `synclink` 的配置值。

```bash
synclink config get <attribute>
synclink config set <attribute> <new_value>
```

**参数：**

*   `get|set`：要执行的操作。
*   `<attribute>`：配置键（例如 `default_sync_path`）。
*   `[new_value]`：仅对于 `set` 需要。属性的新值。

**示例：**

```bash
# 设置链接项目将存储的默认目录
synclink config set default_sync_path C:\Users\You\Dropbox\SyncedStuff

# 获取当前默认同步路径
synclink config get default_sync_path
```

---

## 配置文件

`synclink` 将其配置（包括托管链接和设置的列表）存储在位于可执行文件同目录的文件中：`config.json`

关键设置包括：
*   `default_sync_path`：如果在 `link` 期间未指定 `-s`，用于存储链接项目的根目录。
*   `version`：配置文件版本（当前为 `1.1`），支持自动迁移。

## 配置结构示例

```json
{
  "settings": {
    "default_sync_path": "D:\\Sync"
  },
  "links": {
    "vscode": {
      "shortcut": true,
      "original_path": "C:\\Program Files\\VSCode\\Code.exe",
      "synced_path": "C:\\Users\\You\\AppData\\Roaming\\Microsoft\\Windows\\Start Menu\\Programs\\Visual Studio Code.lnk",
      "description": "Visual Studio Code",
      "created_at": "2025-01-22T10:30:00Z"
    },
    "uv": {
      "shortcut": false,
      "original_path": "C:\\Users\\You\\AppData\\Local\\uv",
      "synced_path": "D:\\Sync\\uv",
      "created_at": "2025-01-20T15:20:00Z"
    }
  },
  "version": "1.1"
}
```

## 新功能说明（v1.1）

### 1. 自定义快捷方式描述

- 使用 `-d` 或 `--description` 参数自定义快捷方式的描述信息
- 如果不指定，程序会自动从 `.exe` 文件的版本信息中读取 `FileDescription`
- 描述信息会存储在配置中，`relink` 时自动恢复

### 2. 自定义快捷方式显示名称

- 使用 `--shortcut-name` 参数指定快捷方式在开始菜单中的显示名称
- 可以使用中文或特殊字符
- 配置名称（`-n`）和显示名称分离，便于管理

### 3. 配置版本迁移

- 自动检测配置版本并执行迁移
- 从旧版本（1.0）升级到新版本（1.1）时无需手动操作
- 保持向后兼容性

## 未来计划 / 路线图

*   **Pointer 命令 (`synclink pointer`)：** *（此功能尚未实现）* 创建小型启动器可执行文件（`pointer.exe`）。这些启动器将包含嵌入信息（如正则表达式路径），在运行时读取以查找和启动实际的目标可执行文件，可能允许更复杂或相对路径的场景，其中简单的快捷方式或符号链接不足够。

## 贡献

欢迎贡献！请随时在 [GitHub 仓库](https://github.com/UkiyoDevs/synclink) 上提交 Pull Request 或开启 Issue。

## 许可证

本项目根据 MIT 许可证授权 - 有关详细信息，请参阅 [LICENSE](LICENSE) 文件。
