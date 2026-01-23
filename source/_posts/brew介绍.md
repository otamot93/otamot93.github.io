---
title: brew介绍
date: 2026-01-23 20:14:25
tags:
    - 工具
    - brew
---

# 介绍

## brew是什么
Homebrew 是 macOS（和 Linux）上最流行的包管理器。没有HomeBrew之前，需要在AppStore、软件官网下载.img或者源码编译安装软件，有了HomeBrew之后，只需要执行`brew install git`进行安装，`brew upgrade git `进行更新,`brew uninstall git`进行卸载。
<!-- more -->

![brew介绍](brew介绍/brew.png)
## 基本概念
Homebrew = 自酿啤酒 🍺
 命名灵感来自"自己酿造"的概念:
### Formula（配方）
命令行软件的安装脚本(Ruby文件)，定义如何下载、编译、安装

### Cask（木桶）
GUI应用程序的安装脚本，用于安装.app,.pkg,.dmg

### Bottle（瓶装）
预编译的二进制包，加速安装

### Cellar（酒窖）
 软件的实际安装位置 (/opt/homebrew/Cellar 或 /usr/local/Cellar)

### Tap（龙头）
第三方仓库，扩展Homebrew的软件源

### Pour（倒酒）
安装bottle
某个软件的特定版本
### Keg（酒桶）
Cellar 中某个软件的特定版本目录

## 核心价值
- **统一管理**: 所有软件在一个地方管理
- **版本控制**: 轻松切换、回退版本
- **依赖管理**: 自动安装依赖项
- **干净卸载**: 完整移除，没有残留
- **社区维护**: 开源，软件包及时更新


## 目录结构
Intel芯片的目录结果如下,Apple Silicon的目录可能有些许不同:
```
  /usr/local/                        # brew --prefix
  ├── Cellar/                        # formula 安装位置
  ├── Caskroom/                      # cask 安装位置
  ├── bin/
  ├── opt/
  └── Homebrew/                      # brew --repository
      ├── Library/
      │   └── Taps/                  # ← tap 在这里！
      ├── docs/
      ├── README.md
      └── ...
```

## 常用命令
```
装与卸载

  brew install <formula>           # 安装 CLI 工具
  brew install --cask <cask>       # 安装 GUI 应用
  brew uninstall <name>            # 卸载
  brew reinstall <name>            # 重新安装

查询与搜索

  brew search <keyword>            # 搜索软件
  brew info <name>                 # 查看软件详情
  brew list                        # 列出已安装的 formula
  brew list --cask                 # 列出已安装的 cask
  brew deps <name>                 # 查看依赖
  brew uses <name>                 # 查看被哪些软件依赖

更新与升级

  brew update                      # 更新 Homebrew 本身
  brew upgrade                     # 升级所有软件
  brew upgrade <name>              # 升级指定软件
  brew outdated                    # 查看可升级的软件

版本管理

  brew pin <name>                  # 锁定版本，禁止升级
  brew unpin <name>                # 解锁
  brew list --pinned               # 查看已锁定的软件
  brew list --versions <name>      # 查看已安装的版本

Tap 管理

  brew tap                         # 列出所有 tap
  brew tap <user/repo>             # 添加 tap
  brew untap <user/repo>           # 移除 tap
  brew tap-new <user/repo>         # 创建新 tap
  brew tap-info <user/repo>        # 查看 tap 信息

清理与维护

  brew cleanup                     # 清理旧版本和缓存
  brew cleanup -n                  # 预览将清理的内容
  brew autoremove                  # 删除不再需要的依赖
  brew doctor                      # 诊断问题
  brew missing                     # 检查缺失的依赖

服务管理

  brew services list               # 列出所有服务
  brew services start <name>       # 启动服务
  brew services stop <name>        # 停止服务
  brew services restart <name>     # 重启服务
  brew services run <name>         # 运行（不设为开机启动）

路径与配置

  brew --prefix                    # Homebrew 安装路径
  brew --repository                # Homebrew 仓库路径
  brew --cellar                    # Cellar 路径
  brew --caskroom                  # Caskroom 路径
  brew --cache                     # 缓存路径
  brew config                      # 查看配置

调试与开发

  brew edit <name>                 # 编辑 formula/cask
  brew cat <name>                  # 查看 formula/cask 内容
  brew home <name>                 # 打开软件官网
  brew fetch <name>                # 只下载不安装
  brew audit <name>                # 检查 formula 规范

实用组合

  # 查看占用空间最大的软件
  brew list --formula | xargs brew info | grep -E "^\S+:|[0-9.]+ [KMG]B" | head -40

  # 导出已安装列表
  brew bundle dump                 # 生成 Brewfile

  # 从 Brewfile 恢复
  brew bundle                      # 安装 Brewfile 中的所有软件
```

# 使用tap自定义软件版本
场景是我需要安装指定的claude-code版本，通过自定义tap，利用官方的ruby模板实现Cask

## 1. 创建 tap
  ```
  brew tap-new local/claude
  ```

  自动生成的结构：
  ```
    homebrew-claude/
  ├── .git/
  ├── .github/
  │   └── workflows/
  │       └── tests.yml
  ├── Formula/              # 用于 formula
  ├── README.md
  └── ...
  ```
不会自动创建Casks目录

## 2. 手动添加 Casks 目录
```
 mkdir $(brew --repository)/Library/Taps/local/homebrew-claude/Casks
```

 
## 3. 放入 cask 文件
```
	vim $(brew --repository)/Library/Taps/local/homebrew-claude/Casks/
```

我的模板是
``` ruby
cask "claude-code" do
  arch arm: "arm64", intel: "x64"
  os macos: "darwin", linux: "linux"

version "2.1.14"
  sha256 arm:          "9f1b058a134ceacb3d972fd07fdb36a4bd9fb01ce8b498e9dc4cc97593689602",
         x86_64:       "0b44e49b28755dc3e42e7bdd20e9a91da79c78b1bd216141f06f96792d31b23c",
         x86_64_linux: "2b31c53e861175a9e4ee8b94653be9fa193718c32b184cb47ad16ea4adc7a1ae",
         arm64_linux:  "36b5586cbf1e1cfafe1b5eed626c10fd0217c3430ecef87dbf32f5b17906960b"

  url "https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases/#{version}/#{os}-#{arch}/claude",
      verified: "storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases/"
  name "Claude Code"
  desc "Terminal-based AI coding assistant"
  homepage "https://www.anthropic.com/claude-code"

  livecheck do
    url "https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases/latest"
    regex(/^v?(\d+(?:\.\d+)+)$/i)
  end

  binary "claude"

  zap trash: [
    "~/.cache/claude",
    "~/.claude",
    "~/.claude.json*",
    "~/.config/claude",
    "~/.local/bin/claude",
    "~/.local/share/claude",
    "~/.local/state/claude",
    "~/Library/Caches/claude-cli-nodejs",
  ]
end

```
## 4. 安装
```
  brew install --cask local/claude/claude-code
```

