---
author: Bobby Grayson
author_link: https://github.com/notactuallypagemcconnell
categories: general
date: 2018-10-01
layout: post
title:  Agnostic Version Management With asdf
excerpt: >
  Take a dive into flexible version management of Elixir, Erlang, and OTP with `asdf`!
---

# 使用 asdf 进行不可知版本管理
## 它是啥？

很多时候我们需要使用多个版本的工具。

很多社区都有自己的东西来做这个事情。

在 Ruby 中，我们有 `chruby`、`rbenv`、`rvm` 等等，NodeJS 有 `nvm`。

这些工具可以让我们轻松快速的在某个项目或环境中切换我们使用的工具。

今天将要讨论我最喜欢的版本管理器 `asdf`，为什么是我最喜欢的呢？因为它可以让您仅用一种工具就可以管理多种语言，因为它与您使用其管理的版本无关。

在我看来，`asdf` 有然而其他工具都没有的一个很大的优点，它让我如此轻松地做到：控制我的 Elixir 是用哪个版本的 OTP 编译的，并把它和 Elixir + OTP 的几个版本一起管理。

让我们来看看它吧!
## 安装
安装 `asdf` 轻而易举。

首先，将其克隆：

```shell
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.5.1
```

现在是时候安装了

在 macOS 上:

```shell
echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bash_profile
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bash_profile
```

在 linux 上(用一个标准的 bash shell):

```shell
echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc
```

使用 ZSH:

```shell
echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.zshrc
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.zshrc
```

使用 Fish:

```shell
echo 'source ~/.asdf/asdf.fish' >> ~/.config/fish/config.fish
mkdir -p ~/.config/fish/completions; and cp ~/.asdf/completions/asdf.fish ~/.config/fish/completions
```
Now restart your shell, and type `asdf` and we get our first introduction to the tool.

现在重启你的 shell，然后输入 `asdf`，我们首先得到一个工具介绍。

```shell
asdf

MANAGE PLUGINS
  asdf plugin-add <name> [<git-url>]   Add a plugin from the plugin repo OR, add a Git repo
                                       as a plugin by specifying the name and repo url
  asdf plugin-list                     List installed plugins
  [...]

MANAGE PACKAGES
  asdf install <name> <version>        Install a specific version of a package or,
                                       with no arguments, install all the package
                                       versions listed in the .tool-versions file
  asdf uninstall <name> <version>      Remove a specific version of a package
  asdf current                         Display current version set or being used for all packages
  asdf current <name>                  Display current version set or being used for package
  [...]

UTILS
  asdf reshim <name> <version>         Recreate shims for version of a package
  asdf update                          Update asdf to the latest stable release
  asdf update --head                   Update asdf to the latest on the master branch
```

## 搭配 Elixir 使用

要让 `asdf` 与 Elixir 一起工作，我们首先需要 Erlang。

根据我们的系统不同，需要有一些简单的步骤。

在 OSX 上:

```shell
brew install autoconf wxmac
asdf plugin-add erlang https://github.com/asdf-vm/asdf-erlang.git
asdf install erlang 21.1
```

在 Ubuntu 上：

```shell
apt-get -y install build-essential autoconf m4 libncurses5-dev libwxgtk3.0-dev libgl1-mesa-dev libglu1-mesa-dev libpng3 libssh-dev
asdf plugin-add erlang https://github.com/asdf-vm/asdf-erlang.git
asdf install erlang 21.1
```

对于 Erlang 的部分，我们可以使用 git 中的任何 ref，或者传递一个主要的 OTP 版本。

`asdf install erlang ref:master` 会让我们从 git 获得最新的 master 版本。

既然我们也可以使用 Elixir 做到这一点，您可以想象它使从特定分支或版本进行构建来调试对 Elixir 本身的贡献（包括多个版本）变得多么容易！

现在，让我们把 Elixir 设置好。

自从我们使用 Erlang 完成管道以来，所有系统上的情况都是相同的。

```shell
asdf plugin-add elixir https://github.com/asdf-vm/asdf-elixir.git
asdf install elixir 1.7
```

现在，如果我们碰巧知道我们需要在 OTP 20 而不是 OTP 21 上进行编译并在该环境中运行，该怎么办？

```shell
asdf install erlang 20.3
asdf install elixir 1.7-otp-20
```

现在，我们可以在给定项目（本地环境，每个目录）中设置要使用的版本，如下所示：

```shell
asdf local erlang 20.3
asdf local elixir 1.7.0-otp-20
```

或者，我们也可以选择设置全局配置（我们的整个系统）：

```shell
asdf global erlang 20.3
asdf global elixir 1.7.0-otp-20
```

要了解有关 asdf 如何管理这些内容并进一步进行自定义的更多信息，请查看[文档](https://github.com/asdf-vm/asdf#the-tool-versions-file)。

如您所见，这使得能够在服务之下有些复杂的工具集之间无缝切换。

我发现 asdf 是管理日常生活中这种复杂性的好工具。

骇客入侵！