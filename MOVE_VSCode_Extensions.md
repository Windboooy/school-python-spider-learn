# 将 VS Code 扩展目录从 C 盘迁移到 D 盘 — 完整操作手册

本文档汇总了在 Windows（PowerShell / CMD）环境下，将 VS Code 扩展目录从默认位置（`%USERPROFILE%\.vscode\extensions`）移动到 D 盘的完整步骤、命令说明、常见失败原因与对应解决方案，以及回滚方法。

**重要说明（执行前必读）**
- 请务必关闭所有 VS Code 窗口后再操作（避免文件被占用）。
- 强烈建议先做备份（压缩或复制目录）。
- 本教程使用目录联接（junction，`mklink /J`），对 VS Code 来说是透明的：程序仍访问原路径，但实际内容在 D 盘。

---

## 1. 环境检查与备份

1. 打开 PowerShell（非管理员即可，但若遇权限问题可使用管理员 PowerShell）。
2. 查看当前扩展目录并列出内容：

```powershell
$old = Join-Path $env:USERPROFILE '.vscode\extensions'
Write-Output "当前扩展目录：$old"
Get-ChildItem -Path $old -Force
```

3. 备份（推荐压缩为 zip）：

```powershell
# 在 D 盘创建备份目录
New-Item -ItemType Directory -Path 'D:\vscode-backup' -Force
# 将扩展目录打包为 zip（可能需要几分钟）
Compress-Archive -Path "$old\*" -DestinationPath "D:\vscode-backup\vscode-extensions-backup.zip"
```

> 备份完成后请确认 `D:\vscode-backup\vscode-extensions-backup.zip` 存在且大小合理。

---

## 2. 在 D 盘创建目标目录

```powershell
# 创建目标目录
$new = 'D:\vscode-extensions'
New-Item -ItemType Directory -Path $new -Force
Get-ChildItem -Path $new -Force
```

说明：`-Force` 会在目录已存在时不报错。

---

## 3. 将扩展移动到 D 盘

在确认 VS Code 已关闭后执行：

```powershell
# 将原目录下所有内容移动到新目录
Get-ChildItem -Path $old -Force | Move-Item -Destination $new

# 确认移动完成
Get-ChildItem -Path $new -Force
Get-ChildItem -Path $old -Force  # 原目录现在应为空或不存在
```

说明：`Move-Item` 会移动文件和文件夹。若遇到“文件被占用”错误，请确保 VS Code 已完全退出（可用任务管理器查看）。

---

## 4. 在原位置创建目录联接（junction）

目录联接会让程序访问原路径时被透明重定向到 D 盘目标。

先确保原路径不存在或为空：

```powershell
# 若原目录仍然存在且为空，可删除
Remove-Item -Path $old -Force -Recurse
```

然后用 `mklink /J` 创建联接：

```powershell
# 在 PowerShell 中调用 cmd 创建 junction
cmd /c "mklink /J "%USERPROFILE%\.vscode\extensions" "D:\vscode-extensions""
```

或直接在管理员 CMD 中运行（如果你已经以管理员打开 CMD）：

```cmd
mklink /J "%USERPROFILE%\.vscode\extensions" "D:\vscode-extensions"
```

创建成功后，访问 `%USERPROFILE%\.vscode\extensions` 会被重定向到 `D:\vscode-extensions`。

---

## 5. 启动 VS Code 并验证

启动 VS Code，检查扩展是否列在“已安装扩展”中并能正常使用。

也可用命令行列出扩展：

```powershell
# 列出已安装扩展
code --list-extensions
```

如果你想临时使用新目录而不创建联接：

```powershell
code --extensions-dir "D:\vscode-extensions"
```

---

## 可选方案（不创建联接）

- 方案 A（临时）: 每次用 `code --extensions-dir "D:\vscode-extensions"` 启动。
- 方案 B（快捷方式）: 修改 VS Code 快捷方式的目标，添加参数 `--extensions-dir "D:\vscode-extensions"`。

注意：有些集成（比如某些系统服务或外部工具）仍可能使用默认路径，联接方案更“透明”。

---

## 常见失败情况与解决方案

1. mklink 报错 “You do not have sufficient privilege to perform this operation”
   - 说明：需要管理员权限。解决：以管理员身份运行 CMD，或在 PowerShell 中 `Start-Process cmd -Verb RunAs` 以管理员打开命令提示符，然后执行 `mklink`。

2. Move-Item 报错“被其他进程使用”或“拒绝访问”
   - 说明：VS Code 或其他程序仍在使用扩展文件。解决：关闭 VS Code、相关辅助程序（如 `Code Helper`、终端中的 `code` 进程），或在任务管理器中结束相关进程，再重试。

3. 联接创建成功但扩展丢失或异常
   - 排查：确认 `D:\vscode-extensions` 中确有扩展文件；检查 `code --list-extensions` 输出。
   - 可能原因：移动过程中某些文件丢失或权限不一致；解决：用备份恢复或重新安装扩展。

4. 跨文件系统硬链接失败（警告像 “Failed to hardlink files; falling back to full copy”）
   - 说明：硬链接与不同分区/文件系统有关。使用 junction（`mklink /J`）而非硬链接不会受此限制。

5. `mklink` 报错“Cannot create a file when that file already exists”
   - 说明：目标路径已存在非空目录或联接。解决：先移动/删除原目录（已备份情况下），再创建联接。

6. 权限导致扩展无法被 VS Code 读取
   - 解决：确保目标目录及其文件具有用户读写权限；必要时在 PowerShell 中运行：

```powershell
# 授予当前用户对 D:\vscode-extensions 的完全控制（示例）
$acl = Get-Acl -Path 'D:\vscode-extensions'
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule($env:USERNAME, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl -Path 'D:\vscode-extensions' -AclObject $acl
```

---

## 回滚方案（若迁移后出现问题）

1. 关闭 VS Code。
2. 删除联接（这只删除联接，不影响 D 盘内容）：

```powershell
Remove-Item -Path "$env:USERPROFILE\.vscode\extensions" -Force
```

3. 将内容从 D 盘移动回原目录：

```powershell
# 重新创建原目录
New-Item -ItemType Directory -Path "$env:USERPROFILE\.vscode\extensions" -Force
Get-ChildItem -Path 'D:\vscode-extensions' -Force | Move-Item -Destination "$env:USERPROFILE\.vscode\extensions"
```

4. 启动 VS Code，确认扩展恢复正常。若使用备份，可直接解压备份文件覆盖原目录。

---

## 验证小清单（迁移完成后）
- [ ] 运行 `code --list-extensions` 能列出扩展
- [ ] 在 VS Code 扩展面板能看到并启用扩展
- [ ] 常用扩展功能正常（例如代码高亮、调试器、语言服务）

---

## 额外提示
- 如果你的工作环境包含 WSL、Docker 或企业策略，建议先在非关键机器或虚拟机上测试迁移流程。
- 若你有多个用户账户，需要为每个用户单独迁移其 `%USERPROFILE%\.vscode\extensions`。

---

## 我可以为你做的事情
- 我可以在当前终端里按顺序执行上述命令（需要你确认并保证已关闭 VS Code，以及是否允许我执行文件移动操作）。
- 或者我只把命令和步骤贴给你，你手动在本地执行（更安全）。

请告诉我你希望我“执行命令”还是“仅生成并离开命令供你手动执行”。
