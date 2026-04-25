---
title: 在macOS M4 上从源码编译ClickHouse 并启动
date: 2026-04-25 11:16:33
tags:
    - ClickHouse
---

  
本文档整理了一个在 `macOS` Apple Silicon（以 `M4` 为例）上，从下载 `ClickHouse` 源码、补齐依赖、完成编译，到启动编译产物的可复现流程。

<!-- more -->

## 1. 前置条件

- 系统：`macOS`（Apple Silicon，`arm64`）
- 包管理器：`Homebrew`
- 本文默认仓库根目录为 `ClickHouse/`

## 2. 安装依赖

先安装 `Homebrew` 依赖：

```bash

brew update

brew install ccache cmake ninja libtool gettext llvm lld binutils grep findutils nasm bash rust rustup

```

安装项目当前要求的 `Rust` 工具链：

```bash

rustup toolchain install nightly-2026-03-22

rustup component add --toolchain nightly-2026-03-22 rust-src

```

可选验证：

```bash

rustup run nightly-2026-03-22 rustc --version

rustup component list --toolchain nightly-2026-03-22 | grep '^rust-src.*installed'

```

## 3. 下载源码

推荐直接带子模块克隆：

```bash

git clone --recurse-submodules https://github.com/ClickHouse/ClickHouse.git

cd ClickHouse

```

如果已经做了普通克隆，或者中途子模块拉取失败，可以继续执行下面的子模块补齐步骤。

## 4. 拉取并修复子模块
  

在 `macOS` 上，`./contrib/update-submodules.sh` 会用到 `grep -P` 和 GNU `find`。因此需要把 `Homebrew` 的 `grep`/`findutils` 放到 `PATH` 前面：

```bash

PATH="$(brew --prefix grep)/libexec/gnubin:$(brew --prefix findutils)/libexec/gnubin:$PATH" ./contrib/update-submodules.sh --max-procs 8

```

说明：

- `--max-procs 8` 是一个相对稳妥的并发值，网络不稳定时比默认高并发更容易成功
- 如果你直接运行脚本并看到 `grep: invalid option -- P`，说明用到了系统自带的 BSD `grep`，重新用上面的命令执行即可

### 4.1 可选：修复“只有 `.git` 指针、没有实际文件”的子模块工作树

如果后续 `cmake` 报下面这类错误：
- `Submodules are not initialized`
- `contrib/sysroot/README.md` 不存在
- `contrib/cctz/testdata/version` 无法读取

通常不是子模块提交不对，而是**部分子模块只有 `.git` 指针，工作树没有 materialize 出来**。可以执行下面的脚本，把这类空工作树补 checkout 出来：

```bash
python3 - <<'PY'
from pathlib import Path
import subprocess

repo = Path.cwd()
out = subprocess.check_output(
    ['git', '-C', str(repo), 'config', '--file', '.gitmodules', '--get-regexp', 'path'],
    text=True,
)

for line in out.splitlines():
    _, rel = line.split(' ', 1)
    rel = rel.strip()
    p = repo / rel
    if not p.exists():
        continue
    entries = sorted(x.name for x in p.iterdir())
    if entries == ['.git']:
        subprocess.run(
            f'git -C "{p}" read-tree HEAD && git -C "{p}" checkout-index -a -f',
            shell=True,
            check=True,
        )
PY
```

执行后，可以用下面的命令确认子模块状态：

```bash
git submodule status
```

理想情况下，每一行前面都应该是空格，而不是 `-` 或 `+`。

## 5. 生成构建配置

`ClickHouse` 在 `macOS` 上要求使用 `Homebrew` 的 `Clang`。先把 `llvm` 放到 `PATH` 前面，再执行 `cmake`：

```bash

PATH="$(brew --prefix llvm)/bin:$PATH" cmake -S . -B build

```

如果这里报 `Cannot find nightly-2026-03-22 Rust toolchain`，回到第 2 步安装对应的 `Rust` nightly。
## 6. 编译

推荐先只编译主程序 `clickhouse`：

```bash

cmake --build build --target clickhouse 2>&1 | tee build/build_clickhouse.log

```

说明：

- 不建议手工加 `-j`，让构建系统自己决定并行度

- 成功后主二进制在 `build/programs/clickhouse`

可选验证：

```bash

./build/programs/clickhouse local --query "SELECT version()"

```

  

## 7. 启动编译后的程序

### 7.1 用独立运行目录启动 `server`

源码树里自带了开发用配置文件：
- `programs/server/config.xml`
- `programs/server/users.xml`
- `programs/server/config.d/path.xml`
- `programs/server/config.d/log_to_console.xml`

其中 `path.xml` 把数据目录、`tmp`、`user_files` 等设置成了相对路径 `./`。因此，推荐在一个单独目录里启动，这样运行时产生的文件不会散落在仓库根目录：

```bash
mkdir -p build/run-local
cd build/run-local
../programs/clickhouse server -C ../../programs/server/config.xml
```

这会在当前目录下生成运行时目录，例如：
- `./tmp/`
- `./user_files/`
- `./access/`
### 7.2 用 `client` 连接

另开一个终端，在仓库根目录执行：
```bash
./build/programs/clickhouse client --host 127.0.0.1
```

连接后可以执行：

```sql
SELECT version();
SELECT 1;
```

如果本机已经有别的 `ClickHouse` 在占用 `9000` / `8123` 端口，需要先停掉它，或者修改 `programs/server/config.xml` 里的端口配置。
### 7.3 停止服务

前台运行时，直接在 `server` 终端按 `Ctrl+C` 即可。

## 8. 常见问题

### 8.1 `grep: invalid option -- P`


原因：脚本用了 `grep -P`，但 `macOS` 自带的是 BSD `grep`。

解决：

```bash

PATH="$(brew --prefix grep)/libexec/gnubin:$(brew --prefix findutils)/libexec/gnubin:$PATH" ./contrib/update-submodules.sh --max-procs 8

```

### 8.2 `Submodules are not initialized`

原因通常不是 superproject 记录错误，而是子模块工作树没有实际 checkout 出来。

解决：执行第 4.1 节的修复脚本，再重新运行：

```bash

PATH="$(brew --prefix llvm)/bin:$PATH" cmake -S . -B build

```

### 8.3 `contrib/cctz/testdata/version cannot be read`

这和上一个问题本质相同，说明 `contrib/cctz` 工作树不完整。执行第 4.1 节的修复脚本即可。

### 8.4 `Cannot find nightly-2026-03-22 Rust toolchain`

执行：

```bash
rustup toolchain install nightly-2026-03-22
rustup component add --toolchain nightly-2026-03-22 rust-src
```
## 9. 一套可直接执行的最短流程

如果你想从头快速跑通，可以按下面的顺序执行：

```bash
brew update
brew install ccache cmake ninja libtool gettext llvm lld binutils grep findutils nasm bash rust rustup

rustup toolchain install nightly-2026-03-22
rustup component add --toolchain nightly-2026-03-22 rust-src

git clone https://github.com/ClickHouse/ClickHouse.git
cd ClickHouse

PATH="$(brew --prefix grep)/libexec/gnubin:$(brew --prefix findutils)/libexec/gnubin:$PATH" ./contrib/update-submodules.sh --max-procs 8

PATH="$(brew --prefix llvm)/bin:$PATH" cmake -S . -B build
cmake --build build --target clickhouse 2>&1 | tee build/build_clickhouse.log

mkdir -p build/run-local
cd build/run-local
../programs/clickhouse server -C ../../programs/server/config.xml
```

另开一个终端连接：

```bash
cd ClickHouse
./build/programs/clickhouse client --host 127.0.0.1
```
