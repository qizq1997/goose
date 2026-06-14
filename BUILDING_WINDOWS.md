# Windows Build Notes

本文记录在 Windows 上跑通本仓库 `cargo build` 的实际经验。当前环境是 PowerShell/CMD，仓库路径为 `C:\codes-internal\goose`。

## 关键结论

- 直接在普通 PowerShell 中运行 `cargo build` 不一定可行，需要先准备 Rust、MSVC、CMake 和 LLVM/libclang。
- 仓库的 `bin/activate-hermit` 是 Bash 脚本；如果 Windows 上没有可用 Bash/WSL，不能直接按 Linux 方式 `source bin/activate-hermit`。
- `bindgen` 需要 `libclang.dll`，本次通过用户目录 LLVM 提供：`C:\Users\qiu\llvm\bin\libclang.dll`。
- `llama-cpp-sys-2` 会编译 C/C++ 依赖，必须加载 Visual Studio Developer Command Prompt 环境。
- 中文 Windows 默认代码页可能导致 MSVC 按 CP936 读取 UTF-8 头文件，需加 `/utf-8`。
- 默认并发编译在本机触发过 Windows `os error 1455`，即页面文件太小；用 `cargo build -j 1` 可以降低峰值内存并成功完成。

## 已验证工具链

- Rust: `rustc 1.92.0`
- Cargo: `cargo 1.92.0`
- MSVC: Visual Studio 2022 Enterprise
- Windows SDK: `10.0.26100.0`
- LLVM/libclang: 安装到 `C:\Users\qiu\llvm`
- CMake: 使用 VS2022 自带 CMake，而不是 VS2019 自带旧版 CMake

## 一次性环境准备

如果当前用户没有 Rust，可安装 rustup 并让仓库的 `rust-toolchain.toml` 选择 Rust `1.92`：

```powershell
$installer = Join-Path $env:TEMP 'rustup-init.exe'
Invoke-WebRequest -Uri 'https://win.rustup.rs/x86_64' -OutFile $installer
& $installer -y --default-toolchain 1.92 --profile default
```

如果缺少 `libclang.dll`，可安装 LLVM 到用户目录，例如：

```powershell
$llvmExe = Join-Path $env:TEMP 'LLVM-22.1.7-win64.exe'
Invoke-WebRequest -Uri 'https://github.com/llvm/llvm-project/releases/download/llvmorg-22.1.7/LLVM-22.1.7-win64.exe' -OutFile $llvmExe
& $llvmExe /S /D=$env:USERPROFILE\llvm
```

安装后确认：

```powershell
Get-ChildItem "$env:USERPROFILE\llvm" -Recurse -Filter libclang.dll
```

## 成功构建命令

在仓库根目录运行：

```bat
chcp 65001 > nul
call "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\Tools\VsDevCmd.bat" -arch=x64 -host_arch=x64
set "LIBCLANG_PATH=C:\Users\qiu\llvm\bin"
set "PATH=C:\Users\qiu\.cargo\bin;C:\Users\qiu\llvm\bin;C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;%PATH%"
set "CMAKE=C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"
set "CXXFLAGS=/utf-8"
set "CFLAGS=/utf-8"
set "CARGO_BUILD_JOBS=1"
cargo build -j 1
```

本次最终结果：

```text
Finished `dev` profile [unoptimized + debuginfo] target(s) in 2m 33s
```

## 常见失败与处理

### cargo/rustc 不在 PATH

现象：

```text
cargo : The term 'cargo' is not recognized
rustc : The term 'rustc' is not recognized
```

处理：

```powershell
$env:Path="$env:USERPROFILE\.cargo\bin;$env:Path"
cargo --version
rustc --version
```

### bindgen 找不到 libclang

现象：

```text
Unable to find libclang: couldn't find any valid shared libraries matching: ['clang.dll', 'libclang.dll']
```

处理：

```bat
set "LIBCLANG_PATH=C:\Users\qiu\llvm\bin"
set "PATH=C:\Users\qiu\llvm\bin;%PATH%"
```

### MSVC 环境未加载

现象通常是 C/C++ crate 找不到编译器、Windows SDK 或 `LIB`/`INCLUDE` 为空。

处理：

```bat
call "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\Tools\VsDevCmd.bat" -arch=x64 -host_arch=x64
```

### llama-cpp-sys-2 编译 UTF-8 头文件失败

现象：

```text
warning C4819: 该文件包含不能在当前代码页(936)中表示的字符
error C2001: 常量中有换行符
```

处理：

```bat
chcp 65001 > nul
set "CXXFLAGS=/utf-8"
set "CFLAGS=/utf-8"
```

### CMake 选到 VS2019 旧版

现象：

```text
CMake Error: Could not create named generator Visual Studio 17 2022
```

原因是 PATH 中先命中了 VS2019 自带 CMake。处理方式是显式使用 VS2022 CMake：

```bat
set "PATH=C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;%PATH%"
set "CMAKE=C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"
```

如果已经生成过失败缓存，可以清理对应 crate 后重试：

```bat
cargo clean -p llama-cpp-sys-2
```

### 页面文件太小或并发导致元数据读取失败

现象：

```text
failed to mmap file ... 页面文件太小，无法完成操作。 (os error 1455)
found invalid metadata files for crate
STATUS_STACK_BUFFER_OVERRUN
```

处理：

```bat
set "CARGO_BUILD_JOBS=1"
cargo build -j 1
```

也可以增加 Windows 页面文件大小，但本次没有修改系统页面文件，单线程构建已经通过。

## 当前编译警告

构建成功时仍存在若干 warning，主要是未使用 import、变量和函数，例如：

- `crates\goose\src\agents\extension_manager.rs` 中未使用 `HeaderValue`
- `crates\goose\src\agents\platform_extensions\developer\shell.rs` 中未使用 `use_login_shell_path`
- `crates\goose-mcp\src\subprocess.rs` 中未使用 `OnceLock`
- `crates\goose-cli\src\commands\session.rs`、`tui.rs`、`update.rs` 中有未使用项

这些 warning 不阻塞 `cargo build`，但后续运行 `cargo clippy --all-targets -- -D warnings` 时需要处理。
