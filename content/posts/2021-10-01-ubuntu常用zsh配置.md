---
title: Ubuntu常用zsh配置
date: 2021-10-01T13:41:22+00:00 
categories:
  - 编程
---

## 一、Install Zsh

```shell
sudo apt install zsh
```

## 二、Install oh-my-zsh

```shell
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

如果提示网络问题可设置socks5代理：

```shell
export all_proxy=socks5://ip:port
```

![安装oh-my-zsh示例](https://objects.wangaiguo.com/wp-content/uploads/2021/10/Screenshot-from-2021-10-01-10-48-07.png)

## 三、安装常用插件

当git无法clone项目时，可单独设置代理：

```shell
git config --global http.proxy http://ip:port  # 设置http全局代理
git config --global https.proxy http://ip:port # 设置https全局代理

git config --global --unset http.proxy  # 取消代理
git config --global --unset https.proxy
```

### zsh-autosuggestions

1. 克隆项目到 oh-my-zsh 的 plugins 目录：

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

2. 在 ~/.zshrc 中激活插件：

```shell
plugins=(git zsh-autosuggestions)
```

3. 重载zsh：

```shell
source ~/.zshrc
```

键入命令时会自动补全历史记录。

![zsh-autosuggestions示例](https://objects.wangaiguo.com/wp-content/uploads/2021/10/Screenshot-from-2021-10-01-11-00-11.png)

### zsh-syntax-highlighting

```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

同样在 .zshrc 中添加插件，重载后即可生效。
