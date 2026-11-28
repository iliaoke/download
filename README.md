# GitHub Actions 远程下载中转

利用 GitHub Actions 的云端网络环境下载对本地网络要求较高的文件，自动处理大文件分卷后发布到 **GitHub Releases**，再通过 GitHub 高速下载到本地。

全程在 GitHub 网页端触发与下载，**无需任何本地脚本或额外客户端**。

---

## 核心特性

- **多协议下载**：基于 `aria2c`，支持 HTTP / HTTPS / FTP / 磁力链接（Magnet）/ BitTorrent 种子，默认 8 线程断点续传。
- **自动避坑磁力做种**：内置 `--seed-time=0` 与 `--bt-stop-timeout=300`，下载完成立即进入打包阶段，避免长时间挂起。
- **2 GiB 自动分卷**：GitHub Releases 单文件上限为 2 GiB；超过阈值的文件会自动以 **1900 MiB / 卷**、仅存储不压缩（`zip -0`）的方式切分为 `.z01`、`.z02`、...、`.zip`。
- **文件名安全处理**：自动清理路径中的特殊字符并在重名时追加序号，避免上传失败或覆盖。
- **SHA256 完整性校验**：Release 说明中自动附带每个附件的大小与 `SHA256` 校验值，方便本地下载后核对。

---

## 工作流程

```text
[手动触发 GitHub Actions]
        │
        ▼
[Ubuntu Runner 使用 aria2c 下载文件]
        │
        ▼
[检测文件大小] ── < 2 GiB ──► 直接准备上传
        │
     >= 2 GiB
        │
        ▼
[zip -0 按 1900 MiB 分卷打包]
        │
        ▼
[发布到 GitHub Releases（附带文件清单与 SHA256）]
        │
        ▼
[在 Releases 页面下载到本地]
```

---

## 快速开始

1. **Fork 本仓库**（或使用本仓库模板创建一个新仓库）。
   > 如果下载内容涉及隐私或不想公开链接，建议创建 **私有仓库（Private）**。
2. 进入你的仓库，点击顶部 **Actions** 标签页，如提示未启用工作流，点击 **I understand my workflows, go ahead and enable them**。
3. 本工作流已在配置中声明 `permissions: contents: write`，通常无需额外配置 Token。若仓库设置限制了默认权限，可在 **Settings -> Actions -> General -> Workflow permissions** 中确认允许读写权限。

---

## 使用方法

1. 打开仓库的 **Actions** 页面，在左侧选择 **下载** 工作流。
2. 点击右侧 **Run workflow** 按钮，填写参数后点击 **Run workflow** 开始执行：

| 参数 | 是否必填 | 说明 |
| --- | --- | --- |
| `download` | 必填 | 目标下载链接（支持 HTTP/HTTPS/FTP/磁力链接等） |
| `filename` | 可选 | 自定义保存文件名（仅对单文件直链生效，用于解决链接文件名乱码或缺少后缀的问题；留空则自动识别） |
| `title` | 可选 | 自定义 Release 标题，方便在多个历史记录中辨认；留空则默认使用 UTC 时间戳（如 `26.11.28_12.00.00`） |

3. 等待任务运行完成：
   - 可在当前 Action 运行页面的 **Summary** 中直接点击生成的 **Release 链接**；
   - 或直接前往仓库右侧 **Releases** 页面下载对应附件。

---

## 大文件分卷解压指南

当单个文件达到或超过 `2 GiB` 时，工作流会自动生成 `.z01`、`.z02`、... 以及最后一个 `.zip` 文件。

**请务必将同一组的所有分卷下载到同一个文件夹中**，然后按系统选择解压方式：

### Windows

使用 **7-Zip** 或 **WinRAR**，右键打开或解压**最后一个 `.zip` 文件**即可自动合并还原。

### Linux / macOS

在分卷所在目录执行：

```bash
zip -s 0 文件名.zip --out merged.zip
unzip merged.zip
```

### 校验文件完整性（可选）

可在本地计算下载文件的 SHA256，与 Release 页面表格中的校验值比对：

- **Windows (PowerShell)**：`Get-FileHash 文件名 -Algorithm SHA256`
- **Linux**：`sha256sum 文件名`
- **macOS**：`shasum -a 256 文件名`

---

## 限制与注意事项

- **单任务时长**：工作流设置了 `timeout-minutes: 180`（3 小时），避免死链或无种资源长时间占用额度。
- **磁盘空间**：GitHub 托管的 `ubuntu-latest` Runner 可用磁盘空间通常在 **14 GB 左右**；由于大文件分卷过程中需要同时存放原文件与分卷，建议单次下载总大小控制在 **6 GB 以内**。
- **单文件上限**：GitHub Releases 单个附件最大支持 **2 GiB**，本工作流已通过 1900 MiB 分卷自动规避。
- **隐私与清理**：
  - 公开仓库（Public）的 Actions 日志与 Releases 对所有人可见，**请勿在公开仓库中填入包含私人鉴权 Token、Cookie 或敏感信息的下载链接**。
  - 下载到本地后，建议及时在 Releases 页面删除不再需要的历史版本以保持仓库整洁。
- **合规使用**：请遵守 GitHub 服务条款及当地法律法规，仅用于合法资源的下载中转。

---

## 常见问题

**Q：为什么直链下载后文件名是一串乱码或没有扩展名？**  
A：部分网盘或 CDN 直链未在响应头中提供标准文件名。触发工作流时在可选参数 `filename` 中手动指定文件名（例如 `archive.tar.gz`）即可。

**Q：磁力链接一直卡在下载步骤怎么办？**  
A：工作流已配置 `--bt-stop-timeout=300`（连续 5 分钟无下载速度自动终止）。如果频繁失败，通常说明该资源缺少活跃做种节点（Peers）。

**Q：提示 `Resource not accessible by integration` 或上传 Release 失败？**  
A：请检查仓库的 **Settings -> Actions -> General -> Workflow permissions**，确保未禁用工作流的 `contents: write` 权限。
