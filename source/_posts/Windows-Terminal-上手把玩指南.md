---
title: Windows Terminal 上手把玩指南
tags: Windows Terminal
categories: Windows
abbrlink: 644b4cd9
date: 2020-05-30 16:56:33
copyright_author: Jitwxs
---

Windows 平台的终端一直以来的确不好用，被 mac 和 linux 吊着锤。历经了 cmd、powershell、FluentTerminal，微软最新的 `Windows Terminal` 终于算是进入可用状态，今天就来把玩把玩。

## 一、安装

安装前提是 Win10 系统，打开 Windows Store 并搜索 `Windows Terminal` 点击安装就行。我挂着梯子的时候，进不去商店，所以各位自己把握就好。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530154104103.png)

## 二、基本了解

安装完毕后，正常打开即可。如果想要折腾折腾的话，那么主要就得了解下它的配置文件。点击窗口栏的设置按钮，进入配置文件。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530154431390.png)

配置文件的大致布局如下所示，对于我们美化来说，只需要关注 `profiles` 和 `schemes` 两块即可。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530155138893.png)

让我们先牛刀小试下，调整下终端的字体和字体大小。因为我想对所有终端都应用这个配置。所以我写在了 `defaults` 中，如下图所示即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2020053015560247.png)

## 三、使用主题

这里以使用 `Iterm2` 的主题资源为例，[访问这里](https://github.com/mbadolato/iTerm2-Color-Schemes/tree/master/windowsterminal) 提供了许多 `Iterm2` 的主题资源，选择其中一个你喜欢的。以 `ChallengerDeep.json` 这个主题为例，其内容如下：

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530160742843.png)

复制内容，粘贴到 `schemes` 下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530161029311.png)

`"colorScheme" : "ChallengerDeep"` 这行配置用于指定使用的主题。

如果你想对所有终端生效，那么把这行配置加在 `defaults` 中，如果只想对某个终端生效，那么就添加到那个终端的配置中，例如我只对 `cmd` 生效：

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530161038414.png)

至此我们完成了主题的替换，最后给大家推荐一个主题网站：https://terminalsplash.com

## 四、添加 GitBash 终端

在很长一段时间，git bash 都是我 Windows 下的主力命令行工具。因为其兼容许多 Linux 命令的语法，除了部分命令无法执行外，它的确很优秀。但现在既然有了 `Windows Terminal`，就把它们俩给整合一下吧。

成品效果如下所示：

- 在终端列表中新增支持 `Git Bash`
- 将其设置为默认首选的终端

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2020053016185069.png)

### 4.1 添加终端

在 `profile -> list` 中，添加上 GitBash 的配置。

```JSON
{
    "guid": "{4B25BFD9-4962-49AE-8512-BBD336462BAB}",
    "name": "Git Bash",
    "commandline": "C:\\Program Files\\Git\\bin\\bash.exe",
    "hidden": false,
    "tabTitle" : "Git Bash",
    "foreground" : "#DCDCDC",
    "icon" : "C:\\Program Files\\Git\\git.ico",
    "acrylicOpacity" : 0.75,
    "backgroundImageOpacity" : 0.25,
    "backgroundImageStretchMode" : "fill",
    "closeOnExit" : true,
    "colorScheme" : "Brogrammer",
    "cursorColor" : "#FFFFFF",
    "cursorShape" : "bar",
    "historySize" : 9001,
    "padding" : "0, 0, 0, 0",
    "snapOnInput" : true,
    "startingDirectory" : "./",
    "useAcrylic" : true
}
```

你需要做出调整的项包括：

- `commandline`: GitBash 实际执行文件的位置，找到你 Git 安装目录下的 `bin/bash.exe` 文件即可。
- `icon`: 展示图标，把[这个图片](https://git-scm.com/favicon.ico)下载下来，保存到本地，替换为你自己的路径即可。
- `startingDirectory`: 打开终端后默认的根路径，设置为 `./` 表示当前路径。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530162800525.png)

### 4.2 设置为默认终端

将配置文件中 `defaultProfile` 值替换为 GitBash 的 ID 即可，这样每次打开 ` Windows Terminal`，默认使用的就是 GitBash 了。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530163327513.png)

## 五、添加右键菜单

感谢网友提供的 Windows Terminal 图标文件，[点击此处](https://gitee.com/Jioho/img/raw/master/wsl/terminal.ico)下载。

进入 `{用户Home目录}/AppData/Local` 文件夹下，创建文件夹 `Terminal`，并将刚刚下载的图标文件放入其中。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530164106125.png)

创建如下注册表文件，并将其保存为 `addwt.reg`，注意要把其中的两个目录，替换为自己的目录。

```reg
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\Background\shell\wt]
@="Windows Terminal Here"
"Icon"="C:\\Users\\Jitwxs\\AppData\\Local\\Terminal\\terminal.ico"

[HKEY_CLASSES_ROOT\Directory\Background\shell\wt\command]
@="C:\\Users\\Jitwxs\\AppData\\Local\\Microsoft\\WindowsApps\\wt.exe"
```

双击执行该文件，执行完毕后。点击鼠标右键，就可以看到刚刚添加的 `Windows Terminal Here` 了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530164438387.png)

## 六、删除右键菜单

既然有了 `Windows Terminal`，原来的 `Git Bash Here` 这个菜单就没啥用了，可以将其关闭了。

点击桌面左下角开始菜单，输入 `redegit` 进入注册表工具，找到 `计算机\HKEY_CLASSES_ROOT\Directory\Background\shell\git_shell` 项。

默认右侧列表项只有两项，点击鼠标右键 `新建 -> 字符串值`，输入内容 `LegacyDisable` ，即可关闭该菜单。

当然你也可以一不做二不休，直接把整个 `git_shell` 文件夹给删了，但是那样你想恢复就不容易了。

![](.https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200530165013720.png)

看到上图中蓝框的 `wt` 文件夹了吗，这就是我们刚刚添加的 Windows Terminal 菜单，如果你不想要了，直接删掉它就好了。
