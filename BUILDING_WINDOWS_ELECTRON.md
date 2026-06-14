# Windows Electron win32-x64 Package Notes

本文记录本次在 Windows 上用 Electron Forge 打包 goose 图形界面应用的实际步骤和踩坑。最终产物是 win32-x64 的可运行应用目录：

```text
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\Goose.exe
```

包内后端二进制位置：

```text
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources\bin\goosed.exe
```

## 前置条件

需要先完成 Rust 侧构建，至少生成：

```text
C:\codes-internal\goose\target\debug\goose.exe
C:\codes-internal\goose\target\debug\goosed.exe
```

需要 Node 和 pnpm：

```text
node v24.10.0
pnpm 10.30.3
```

注意：本机 PATH 上原本是 `pnpm 11.6.0`，Electron Forge 的 pnpm 检查不兼容，需要切到项目声明的 `pnpm 10.30.3`。

## 安装 UI 依赖

UI 是独立 workspace，lockfile 在 `ui\pnpm-lock.yaml`，需要在 `ui` 目录安装依赖。

```bat
cd C:\codes-internal\goose\ui
corepack pnpm@10.30.3 install --frozen-lockfile
```

如果全局 `pnpm` 版本太新，先激活项目要求的版本：

```bat
corepack prepare pnpm@10.30.3 --activate
pnpm --version
```

确认输出是：

```text
10.30.3
```

## Electron 下载问题

本机依赖安装时 `electron` postinstall 下载 Electron 二进制失败：

```text
RequestError: connect ECONNREFUSED 127.0.0.1:443
```

环境里存在代理变量：

```text
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
ALL_PROXY=socks5://127.0.0.1:7890
```

`curl` 可以正常下载，但 Node 里的下载器会失败。处理方式是手动把 Electron zip 放进 `@electron/get` 缓存目录，然后重跑安装脚本。

缓存路径可用 Node 算出：

```bat
cd C:\codes-internal\goose\ui
corepack pnpm@10.30.3 exec node -e "const {Cache}=require('@electron/get/dist/cjs/Cache'); const {getArtifactRemoteURL,getArtifactFileName,getArtifactVersion}=require('@electron/get/dist/cjs/artifact-utils'); (async()=>{const d={version:getArtifactVersion({version:'41.0.0'}),artifactName:'electron',platform:'win32',arch:'x64'}; const url=await getArtifactRemoteURL(d); const file=getArtifactFileName(d); const c=new Cache(); console.log(url); console.log(c.getCachePath(url,file));})()"
```

本次得到：

```text
https://github.com/electron/electron/releases/download/v41.0.0/electron-v41.0.0-win32-x64.zip
C:\Users\qiu\AppData\Local\electron\Cache\046d2c8217fed3535d6b67e6ada50a34f07c939c8e53e9a4f5c5a9822f65103f\electron-v41.0.0-win32-x64.zip
```

手动下载：

```powershell
$url = 'https://github.com/electron/electron/releases/download/v41.0.0/electron-v41.0.0-win32-x64.zip'
$dest = 'C:\Users\qiu\AppData\Local\electron\Cache\046d2c8217fed3535d6b67e6ada50a34f07c939c8e53e9a4f5c5a9822f65103f\electron-v41.0.0-win32-x64.zip'
New-Item -ItemType Directory -Force -Path (Split-Path $dest) | Out-Null
curl.exe -L $url -o $dest --fail --retry 3 --retry-delay 5
```

然后重跑 Electron 安装：

```bat
cd C:\codes-internal\goose\ui
corepack pnpm@10.30.3 exec node node_modules\electron\install.js
corepack pnpm@10.30.3 install --frozen-lockfile
```

## 准备 Windows 二进制资源

Electron app 运行时会找：

```text
resources\bin\goosed.exe
```

打包前需要把 Rust 产物复制到桌面应用的 `src\bin`：

```powershell
cd C:\codes-internal\goose
New-Item -ItemType Directory -Force -Path ui\desktop\src\bin | Out-Null
Copy-Item -Force target\debug\goose.exe ui\desktop\src\bin\goose.exe
Copy-Item -Force target\debug\goosed.exe ui\desktop\src\bin\goosed.exe
```

安装时还遇到 `@aaif/goose-binary-win32-x64` 包没有 `bin\goose.exe` 的 warning：

```text
Failed to create bin ... @aaif\goose-binary-win32-x64\bin\goose.exe.EXE
```

本次也补了一份本地二进制：

```powershell
New-Item -ItemType Directory -Force -Path ui\goose-binary\goose-binary-win32-x64\bin | Out-Null
Copy-Item -Force target\debug\goose.exe ui\goose-binary\goose-binary-win32-x64\bin\goose.exe
```

这些 `.exe` 和 `out` 目录都被现有 `.gitignore` 忽略，不应提交。

## uv 下载问题

`ui\desktop\scripts\prepare-platform-binaries.js` 会下载 Windows 用的 `uv.exe` 和 `uvx.exe`。本机 Node 下载同样失败：

```text
Downloading uv 0.11.11 ...
Error: connect ECONNREFUSED 127.0.0.1:443
```

处理方式是手动下载并复制：

```powershell
cd C:\codes-internal\goose\ui\desktop
$url = 'https://github.com/astral-sh/uv/releases/download/0.11.11/uv-x86_64-pc-windows-msvc.zip'
$tmp = Join-Path $env:TEMP 'goose-uv-manual'
$zip = Join-Path $tmp 'uv.zip'
$extract = Join-Path $tmp 'extract'
Remove-Item -Recurse -Force $tmp -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force -Path $extract | Out-Null
curl.exe -L $url -o $zip --fail --retry 3 --retry-delay 5
Expand-Archive -LiteralPath $zip -DestinationPath $extract -Force
Copy-Item -Force (Get-ChildItem $extract -Recurse -Filter uv.exe | Select-Object -First 1 -ExpandProperty FullName) src\bin\uv.exe
Copy-Item -Force (Get-ChildItem $extract -Recurse -Filter uvx.exe | Select-Object -First 1 -ExpandProperty FullName) src\bin\uvx.exe
```

校验哈希：

```powershell
Get-FileHash src\bin\uv.exe -Algorithm SHA256
Get-FileHash src\bin\uvx.exe -Algorithm SHA256
```

本次哈希：

```text
uv.exe  B1645E948603C12DD741987D0C072471195E18DD299B42334477CEAC694F0AF8
uvx.exe 0305C488DC29C16DF1483C02A902D21A6798B0744F8E9EB34271D6B3E4BF6E2A
```

随后重跑准备脚本：

```bat
cd C:\codes-internal\goose\ui\desktop
corepack pnpm@10.30.3 exec node scripts\prepare-platform-binaries.js
```

成功输出包含：

```text
Pinned uv 0.11.11 binaries already present
Platform binary preparation complete
```

## pnpm node-linker 问题

Electron Forge package 时失败：

```text
When using pnpm, `node-linker` must be set to "hoisted"
```

虽然 `ui\.npmrc` 已有：

```text
node-linker=hoisted
```

但 Forge 子进程会调用 PATH 上的 `pnpm`。如果 PATH 上是 `pnpm 11.6.0`，它可能读不到项目期望配置。本次处理：

```bat
corepack prepare pnpm@10.30.3 --activate
pnpm --version
pnpm config get node-linker
```

确认：

```text
10.30.3
hoisted
```

本次还新增了 `ui\desktop\.npmrc`：

```text
node-linker=hoisted
```

这样从 `ui\desktop` 目录直接运行 Forge 时也能明确读到配置。

## 成功打包命令

```bat
cd C:\codes-internal\goose\ui\desktop
pnpm run i18n:compile
pnpm exec electron-forge package --platform=win32 --arch=x64
```

成功输出关键行：

```text
Packaging for x64 on win32
Packaging application
```

## 输出检查

确认 GUI exe：

```powershell
Get-ChildItem C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64 -Filter Goose.exe
```

确认包内后端：

```powershell
Get-ChildItem C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources -Recurse -Filter goosed.exe
```

本次输出包括：

```text
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\Goose.exe
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources\bin\goose.exe
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources\bin\goosed.exe
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources\bin\uv.exe
C:\codes-internal\goose\ui\desktop\out\Goose-win32-x64\resources\bin\uvx.exe
```

## package 与 make 的区别

本次执行的是：

```bat
electron-forge package --platform=win32 --arch=x64
```

它生成的是可直接运行的应用目录，不是安装器。如果需要安装包，需要继续运行：

```bat
pnpm exec electron-forge make --platform=win32 --arch=x64
```

`make` 可能会额外触发 maker 下载、签名或安装器生成步骤，和本次 `package` 不完全相同。
