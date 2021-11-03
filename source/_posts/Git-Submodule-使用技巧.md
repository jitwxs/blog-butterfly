---
title: Git Submodule 使用技巧
categories:
  - Dev Tools
  - Git
tags: Git
abbrlink: 8bef605
date: 2019-07-11 12:11:15
---

有的时候我们会遇到仓库嵌套的问题，即一个 Git 仓库内部还有一个 Git 仓库，这里我们可以使用 Git 的模块化。

现在我拥有一个 git 项目 `blog`，它的内部有一个博客主题，名为 `hexo-theme-icarus`，这也是一个 git 项目，这里就可以把这个主题项目作为模块引入进来。

为了方便命令介绍，先大致画一下目录结构：

```ini
- blog(博客项目)
    - aaa
    - bbb
    - theme
        - hexo-theme-icarus(主题项目)
```

**Step1：** 首先进入 theme 目录下，将 hexo-theme-icarus `clone` 下来，这里已经完成了。

**Step2：** 然后就是把它加入到模块中，在 theme 目录下：

```shell
wxs@wxs-PC MINGW32 ~/Documents/GitHub/blog/themes (master)
$ git submodule add https://github.com/jitwxs/hexo-theme-icarus.git hexo-theme-icarus/
Adding existing repo at 'themes/hexo-theme-icarus' to the index
warning: LF will be replaced by CRLF in .gitmodules.
The file will have its original line endings in your working directory.
```

通过 `git submodule add` 添加模块，第一个参数是模块的 git 地址，第二个是模块的文件夹路径。

此时就会生成一个 `.gitmodules` 文件到 blog 项目下，查看它的内容发现列出了 blog 项目下的所有模块的路径及远程的仓库地址：

```shell
wxs@wxs-PC MINGW32 ~/Documents/GitHub/blog/themes (master)
$ cat ../.gitmodules
[submodule "themes/hexo-theme-icarus"]
        path = themes/hexo-theme-icarus
        url = https://github.com/jitwxs/hexo-theme-icarus.git
```

**Step3：** 如果你想删除模块的话，执行 ` git rm -r --cached hexo-theme-icarus/`，并删除对应的 `.gitmodules` 中的模块信息即可。
