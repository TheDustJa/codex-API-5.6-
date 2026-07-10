# OpenAI 官方 ChatGPT Windows 应用无法识别 GPT-5.6：问题分析与修复方法

## 1. 文档目的

本文记录 OpenAI 官方 Windows ChatGPT 应用在使用自定义 API Key 网关时，无法在模型选择器中显示 GPT-5.6 模型的问题，以及一套可验证、可回滚的本地兼容修复方法。

本文针对的是 Windows 开始菜单中显示为 **ChatGPT** 的 OpenAI 官方桌面应用，不是 VS Code 的 ChatGPT/Codex 扩展。

本次排查环境：

- 应用显示名：`ChatGPT`
- MSIX 包名：`OpenAI.Codex`
- 包版本：`26.707.3748.0`
- 包架构：`x64`
- 默认模型：`gpt-5.6-sol`
- 认证方式：自定义网关的 API Key 模式

> 注意：应用版本升级后，资源文件名、哈希值和安装目录都会变化。本文中的具体版本号和资源文件名仅适用于对应版本，修复思路仍可复用。

## 2. 问题现象

自定义网关和 Codex app-server 已经能够正常返回以下模型：

- `gpt-5.6-sol`
- `gpt-5.6-terra`
- `gpt-5.6-luna`

同时，`~/.codex/config.toml` 中的默认模型已经配置为：

```toml
model = "gpt-5.6-sol"
```

但是，OpenAI 官方 ChatGPT Windows 应用的模型选择器仍无法显示 GPT-5.6，或者会把默认模型回退到较旧模型。

这会造成一种容易误判的现象：

- 网关支持 GPT-5.6；
- app-server 能返回 GPT-5.6；
- 命令行直接指定 GPT-5.6 可以工作；
- 只有 ChatGPT 桌面应用的前端不显示该模型。

因此，该问题并不是网关转发失败，也不是 API Key、`base_url` 或模型定价配置错误，而是桌面应用前端对模型列表进行了二次过滤。

## 3. 根因分析

### 3.1 官方应用的实际包身份

可以通过 PowerShell 查看应用包：

```powershell
Get-AppxPackage | Where-Object {
    $_.Name -match 'ChatGPT|OpenAI' -or
    $_.PackageFullName -match 'ChatGPT|OpenAI'
} | Select-Object Name, PackageFullName, Version, InstallLocation
```

本次环境中，开始菜单中的 ChatGPT 实际对应：

```text
Name: OpenAI.Codex
DisplayName: ChatGPT
Version: 26.707.3748.0
```

程序入口由 `AppxManifest.xml` 声明为：

```text
app/ChatGPT.exe
```

桌面应用前端资源封装在：

```text
app/resources/app.asar
```

### 3.2 有问题的过滤逻辑

解包 `app.asar` 后，在以下文件中找到模型列表过滤逻辑：

```text
webview/assets/model-list-filter-C2SM1X_9.js
```

原始逻辑的核心等价于：

```javascript
useHiddenModels && authMethod !== "amazonBedrock"
```

压缩后的实际代码类似：

```javascript
u = s && e !== `amazonBedrock`
```

当 `useHiddenModels` 开启时，代码不会直接显示 app-server 返回的所有 `hidden=false` 模型，而是要求模型同时存在于前端获得的远程 `availableModels` 白名单中。

API Key 账号虽然能从自定义网关获得 GPT-5.6，但 GPT-5.6 尚未出现在 ChatGPT 远程白名单中，所以在桌面应用前端被隐藏。

### 3.3 为什么判断值必须是 `apikey`

app-server 返回的账号类型通常是：

```json
{"type":"apiKey"}
```

但前端内部会将其归一化为小写形式：

```text
apikey
```

因此条件必须判断 `apikey`，不能写成 `apiKey`。错误的大小写会导致补丁条件永远无法命中。

### 3.4 正确的过滤行为

修复后的逻辑应等价于：

```javascript
useHiddenModels &&
authMethod !== "amazonBedrock" &&
authMethod !== "apikey"
```

压缩代码中的修改为：

```javascript
u = s && e !== `amazonBedrock` && e !== `apikey`
```

修改后的行为：

- ChatGPT 登录继续遵守远程模型白名单；
- Amazon Bedrock 保持原有行为；
- 自定义 API Key 网关不再受 ChatGPT 远程模型白名单限制；
- API Key 模式显示 app-server 返回且 `hidden=false` 的模型；
- 不会凭空增加模型，模型仍必须由 app-server 或网关实际返回。

## 4. 为什么不能直接修改 Store 安装目录

官方应用安装在类似以下目录：

```text
C:\Program Files\WindowsApps\OpenAI.Codex_<版本>_x64__2p2nqsd0c76g0
```

该目录具有多层保护：

1. 目录和文件由 `TrustedInstaller` 或系统账户管理；
2. Microsoft Store/MSIX 包具有签名和包完整性保护；
3. 即使临时取得管理员权限并修改单个文件 ACL，系统仍可能拒绝替换包资源；
4. 强行篡改可能导致应用无法启动、Store 修复应用或后续增量更新失败。

本次尝试仅针对单个 `app.asar` 保存原文件、取得临时权限并替换，但 Windows 仍返回访问拒绝。失败处理自动恢复了原始文件与 ACL，官方 Store 版本没有被修改。

因此，不建议通过接管整个 `WindowsApps` 目录、关闭系统保护或删除包签名文件来修复。

## 5. 推荐修复方案：创建本地兼容副本

安全方案是：

1. 保留 Store 安装的官方应用不变；
2. 将应用的 `app` 目录复制到当前用户可写目录；
3. 解包 `app.asar`；
4. 只修改模型过滤条件；
5. 重新封装 `app.asar`；
6. 用补丁包替换本地副本中的 `resources/app.asar`；
7. 创建独立开始菜单入口；
8. 验证补丁哈希和进程启动情况。

这种方式的优点：

- 不破坏原 Store 应用；
- 不需要长期修改 `WindowsApps` 权限；
- 可以随时回退到原版；
- 修复范围明确，仅影响 API Key 模式的模型展示；
- 便于针对新版本重新生成补丁。

缺点：

- 本地副本会额外占用磁盘空间；
- Store 更新不会自动同步到本地副本；
- 每次官方应用升级后，应基于新版本重新解包和打补丁；
- 本地副本不再是由 Store 管理的原始签名包运行路径。

## 6. 详细修复步骤

### 6.1 前置条件

需要：

- Windows PowerShell；
- Node.js；
- 可用的 `asar` CLI；
- 足够的磁盘空间；
- 已安装 OpenAI 官方 ChatGPT Windows 应用。

检查工具：

```powershell
node --version
npx --no-install asar --version
```

如果本机未缓存 `asar`，需要先通过可信的软件包来源安装。不要从未知位置下载修改后的 `asar` 工具。

### 6.2 获取应用安装信息

```powershell
$pkg = Get-AppxPackage OpenAI.Codex
$pkg | Select-Object Name, Version, InstallLocation, PackageFullName
```

设置路径：

```powershell
$sourceApp = Join-Path $pkg.InstallLocation 'app'
$sourceAsar = Join-Path $sourceApp 'resources\app.asar'
$version = $pkg.Version.ToString()
$workDir = Join-Path $env:TEMP "OpenAI.Codex-$version-unpacked"
$patchedAsar = Join-Path $env:TEMP "OpenAI.Codex-$version-patched.asar"
$backupDir = Join-Path $env:LOCALAPPDATA "OpenAI\backups\OpenAI.Codex_$version"
$localApp = Join-Path $env:LOCALAPPDATA "OpenAI\ChatGPT-GPT56\$version"
```

### 6.3 备份原始资源

```powershell
New-Item -ItemType Directory -Path $backupDir -Force | Out-Null
$backupAsar = Join-Path $backupDir 'app.asar.original'

if (!(Test-Path -LiteralPath $backupAsar)) {
    Copy-Item -LiteralPath $sourceAsar -Destination $backupAsar
}

Get-FileHash -Algorithm SHA256 $sourceAsar, $backupAsar
```

两个哈希值必须一致。

### 6.4 解包 app.asar

```powershell
if (Test-Path -LiteralPath $workDir) {
    throw "临时解包目录已存在，请先人工确认后处理：$workDir"
}

npx --no-install asar extract $sourceAsar $workDir
```

查找模型过滤文件：

```powershell
rg -l -S 'useHiddenModels|amazonBedrock|availableModels' `
    $workDir --glob '*.js'
```

不同版本的文件哈希名可能不同，例如：

```text
model-list-filter-C2SM1X_9.js
```

不要依赖固定哈希文件名，应根据代码内容定位。

### 6.5 修改过滤条件

在定位到的 `model-list-filter-*.js` 中，将：

```javascript
u=s&&e!==`amazonBedrock`
```

改为：

```javascript
u=s&&e!==`amazonBedrock`&&e!==`apikey`
```

变量名可能因版本压缩而改变。应以语义为准：

- 第一个变量是 `useHiddenModels`；
- 第二个变量是归一化后的 `authMethod`；
- 必须使用全小写 `apikey`。

修改后进行语法检查：

```powershell
node --check '<解包目录>\webview\assets\model-list-filter-<哈希>.js'
```

### 6.6 重新封装

```powershell
if (Test-Path -LiteralPath $patchedAsar) {
    Remove-Item -LiteralPath $patchedAsar -Force
}

npx --no-install asar pack $workDir $patchedAsar
Get-Item -LiteralPath $patchedAsar
Get-FileHash -LiteralPath $patchedAsar -Algorithm SHA256
```

### 6.7 创建本地应用副本

在复制前确认目标路径位于当前用户的 LocalAppData 中：

```powershell
New-Item -ItemType Directory -Path $localApp -Force | Out-Null

$expectedRoot = (Resolve-Path (Join-Path $env:LOCALAPPDATA 'OpenAI\ChatGPT-GPT56')).Path
$resolvedTarget = (Resolve-Path $localApp).Path

if (!$resolvedTarget.StartsWith($expectedRoot + '\', [StringComparison]::OrdinalIgnoreCase)) {
    throw '目标目录超出了预期的本地应用目录。'
}
```

复制官方应用文件：

```powershell
robocopy $sourceApp $localApp /E /COPY:DAT /DCOPY:DAT /R:1 /W:1

if ($LASTEXITCODE -ge 8) {
    throw "应用复制失败，robocopy exit code: $LASTEXITCODE"
}
```

替换本地副本的资源：

```powershell
$localAsar = Join-Path $localApp 'resources\app.asar'
Copy-Item -LiteralPath $patchedAsar -Destination $localAsar -Force
```

核对哈希：

```powershell
$installedHash = (Get-FileHash -LiteralPath $localAsar -Algorithm SHA256).Hash
$patchHash = (Get-FileHash -LiteralPath $patchedAsar -Algorithm SHA256).Hash

if ($installedHash -ne $patchHash) {
    throw '本地副本的 app.asar 与补丁包哈希不一致。'
}
```

### 6.8 创建开始菜单入口

```powershell
$exe = Join-Path $localApp 'ChatGPT.exe'
$shortcut = Join-Path $env:APPDATA `
    'Microsoft\Windows\Start Menu\Programs\ChatGPT GPT-5.6.lnk'

$shell = New-Object -ComObject WScript.Shell
$link = $shell.CreateShortcut($shortcut)
$link.TargetPath = $exe
$link.WorkingDirectory = $localApp
$link.IconLocation = "$exe,0"
$link.Description = 'OpenAI ChatGPT with GPT-5.6 API-key model compatibility patch'
$link.Save()
```

完成后，从开始菜单启动：

```text
ChatGPT GPT-5.6
```

不要误启动未打补丁的 Store 版 `ChatGPT`。

## 7. 配置要求

确认 Codex 配置的默认模型：

```powershell
Select-String -LiteralPath "$HOME\.codex\config.toml" `
    -Pattern '^\s*model\s*='
```

预期：

```toml
model = "gpt-5.6-sol"
```

该修复不需要修改：

- API Key；
- 自定义网关地址；
- `base_url`；
- 推理等级；
- 认证令牌；
- 原 Store 应用的注册信息。

不要把包含 API Key、访问令牌或内部网关地址的完整配置文件复制到公开问题报告中。

## 8. 验证方法

### 8.1 静态验证

检查补丁代码：

```powershell
rg -n -S 'amazonBedrock.*apikey' `
    '<解包目录>\webview\assets\model-list-filter-*.js'
```

检查 JavaScript 语法：

```powershell
node --check '<模型过滤文件>'
```

检查本地资源哈希：

```powershell
Get-FileHash -Algorithm SHA256 $localAsar, $patchedAsar
```

两个哈希必须一致。

### 8.2 进程验证

启动本地修复副本后：

```powershell
Get-Process -ErrorAction SilentlyContinue | Where-Object {
    $_.Path -like "$env:LOCALAPPDATA\OpenAI\ChatGPT-GPT56\*"
} | Select-Object Id, ProcessName, Path, StartTime
```

正常情况下会看到：

- 一个或多个 `ChatGPT.exe` 进程；
- 本地副本目录中的 `resources\codex.exe` 进程。

### 8.3 功能验证

在应用中确认：

1. 模型列表出现 `GPT-5.6 Sol`、`Terra` 或 `Luna`；
2. 默认模型为 `GPT-5.6 Sol`；
3. 发起普通测试请求能够成功返回；
4. 网关日志中实际请求模型为 `gpt-5.6-sol`；
5. ChatGPT 登录模式的模型白名单行为没有被改变。

## 9. 回滚方法

### 9.1 最简单的回滚

退出 `ChatGPT GPT-5.6`，然后直接使用开始菜单中的原版：

```text
ChatGPT
```

原 Store 应用未被修改，因此不需要修复或重新安装。

### 9.2 删除本地修复副本

确认所有本地副本进程已经退出后，删除：

```text
%LOCALAPPDATA%\OpenAI\ChatGPT-GPT56\<版本>
```

同时删除开始菜单快捷方式：

```text
%APPDATA%\Microsoft\Windows\Start Menu\Programs\ChatGPT GPT-5.6.lnk
```

删除属于破坏性操作，应在确认路径和进程后人工执行。

原始 `app.asar` 备份可继续保留：

```text
%LOCALAPPDATA%\OpenAI\backups\OpenAI.Codex_<版本>\app.asar.original
```

## 10. 应用升级后的处理

Store 更新后：

1. 查询新的 `OpenAI.Codex` 版本；
2. 不要直接复用旧版 `app.asar`；
3. 从新版本的官方 `app.asar` 重新解包；
4. 重新定位新的 `model-list-filter-*.js`；
5. 核对新版本是否已经由官方加入 `apikey` 例外；
6. 如果官方已经修复，应停止使用本地补丁；
7. 如果问题仍存在，再基于新版本生成新的本地副本。

旧版 `app.asar` 不能覆盖到新版应用，因为模块结构、资源哈希、导入关系和 Electron 主进程代码可能已经变化。

## 11. 常见错误

### 11.1 把认证类型写成 `apiKey`

错误：

```javascript
authMethod !== "apiKey"
```

正确：

```javascript
authMethod !== "apikey"
```

前端使用的是归一化后的全小写值。

### 11.2 只修改 config.toml

`config.toml` 决定默认请求模型，但不能解除桌面应用前端的远程白名单过滤。必须同时解决前端过滤问题。

### 11.3 直接修改 WindowsApps

即使管理员取得了文件所有权，MSIX 包保护仍可能拒绝替换。强行递归修改 `WindowsApps` ACL 会扩大风险，不推荐。

### 11.4 直接复用 VS Code 扩展补丁

VS Code 扩展和官方桌面应用是两套独立安装资源：

- VS Code：`.vscode/extensions/openai.chatgpt-*`
- 官方桌面应用：`WindowsApps/OpenAI.Codex_*`

修改其中一个不会自动修复另一个。

### 11.5 忘记重新封装或哈希验证

只修改解包目录不会影响运行中的应用。必须重新 `asar pack`，替换本地副本资源，并核对 SHA-256。

### 11.6 Codex app 无法显示 GPT-5.6 模型

**问题现象：** 在 Codex CLI 或 Codex app 中，模型选择列表里看不到 `gpt-5.6-sol` / `terra` / `luna`，或启动时自动降级到旧模型。

**根因：** 与桌面 ChatGPT 应用相同——前端对模型列表做了二次过滤。`config.toml` 中配置的 `model = "gpt-5.6-sol"` 只影响默认请求模型，不能解除前端白名单限制。

**解决办法：** 将下方提示词完整复制，粘贴给 Codex，让它自动执行修复流程。

<details>
<summary>▶ 点击展开修复提示词</summary>

```
你好，请帮我修复 Codex app 无法显示 GPT-5.6 模型的问题。

背景：
- 自定义 API Key 网关已能返回 gpt-5.6-sol / gpt-5.6-terra / gpt-5.6-luna
- ~/.codex/config.toml 已设置 model = "gpt-5.6-sol"
- 但 Codex 或 ChatGPT 桌面应用的模型选择器仍看不到这些模型

根因：
桌面应用前端在 webview/assets/model-list-filter-*.js 中有如下过滤逻辑：
  u = s && e !== `amazonBedrock`
当 authMethod 为 apikey（全小写）时，仍会应用远程白名单过滤，导致 GPT-5.6 被隐藏。

修复目标：
将过滤条件改为：
  u = s && e !== `amazonBedrock` && e !== `apikey`
使 API Key 模式跳过远程白名单，直接展示 app-server 实际返回且 hidden=false 的模型。

请按以下步骤操作：

1. 查询已安装的 OpenAI.Codex 包信息：
   Get-AppxPackage OpenAI.Codex | Select-Object Name, Version, InstallLocation

2. 设置路径变量（$pkg、$sourceAsar、$workDir、$patchedAsar、$backupDir、$localApp）

3. 备份原始 app.asar，核对 SHA-256 一致

4. 用 `npx --no-install asar extract` 解包到临时目录

5. 用 ripgrep 找到 model-list-filter-*.js：
   rg -l -S "useHiddenModels|amazonBedrock" $workDir --glob "*.js"

6. 将文件中的
     u=s&&e!==`amazonBedrock`
   替换为
     u=s&&e!==`amazonBedrock`&&e!==`apikey`
   （变量名以实际压缩结果为准，语义：useHiddenModels && authMethod !== amazonBedrock && authMethod !== apikey）

7. 用 `node --check` 验证语法

8. 用 `npx --no-install asar pack` 重新封装为 patched.asar

9. 复制官方 app 目录到 %LOCALAPPDATA%\OpenAI\ChatGPT-GPT56\<version>
   robocopy $sourceApp $localApp /E /COPY:DAT /R:1 /W:1

10. 替换本地副本的 resources\app.asar，核对 SHA-256 一致

11. 创建开始菜单快捷方式 "ChatGPT GPT-5.6"

12. 静态验证：
    rg -n -S "amazonBedrock.*apikey" <解包目录>\webview\assets\model-list-filter-*.js

完成后告诉我每步的执行结果和最终 SHA-256 哈希对比。
注意：不要修改 WindowsApps 目录下的原始 Store 应用文件。
```

</details>

**验证：** 修复完成后，从开始菜单启动 `ChatGPT GPT-5.6`，在模型选择器中确认出现 `GPT-5.6 Sol` / `Terra` / `Luna`。

## 12. 安全边界

本修复仅改变 API Key 模式的模型展示过滤行为，不应：

- 修改或记录 API Key；
- 绕过账号权限或模型授权；
- 伪造网关未返回的模型；
- 修改 Windows 安全设置；
- 关闭 Store/MSIX 签名校验；
- 接管整个 `WindowsApps` 目录；
- 将内部网关地址和令牌写入公开日志。

补丁只能让前端展示 app-server 实际返回并允许显示的模型。最终能否调用 GPT-5.6，仍取决于自定义网关、账号权限和上游接口是否真实支持该模型。

## 13. 本次修复结果

本次环境最终采用本地兼容副本方案：

- Store 原版 ChatGPT 保持不变；
- 本地修复副本创建成功；
- `app.asar` 补丁哈希校验通过；
- 开始菜单已创建 `ChatGPT GPT-5.6`；
- 默认模型保持为 `gpt-5.6-sol`；
- 本地 `ChatGPT.exe` 和内置 `codex.exe` 进程启动成功；
- API Key 模式不再被 ChatGPT 远程模型白名单错误限制。

当 OpenAI 官方版本原生修复该问题后，应优先恢复使用 Store 正式版本，并移除本地兼容副本。
