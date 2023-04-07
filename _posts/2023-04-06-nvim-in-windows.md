---
title: 在 Windows 下使用 NeoVim 
date: 2023-04-06 21:58:58 +0800
categories: [nvim]
tags: [nvim]
---

这两年一直在服务器上使用 neovim，最近尝试把它放到 Windows 上用。

一旦习惯图形化界面，就很难第一时间想到完全使用 TUI 工具。

# 启动 nvim 的方式很多

首先是 nvim 的启动问题，只要在终端上工作，启动 nvim 非常自然，但是在 Windows 上体验很不好，选择/流程太多：
* 随 nvim 附带的 `nvim-qt.exe` 是非常好的 GUI 工具，它给你最大的好处是用于关联文件类型。
  直接在资源管理器中双击一个文件，就能用 nvim 打开。有以下一些缺点：
  * 首次使用它，需要配置 [`ginit.vim`](https://github.com/equalsraf/neovim-qt#configuration)
  * 初次打开新的文件类型，需要每次关联
  * GUI 的启动速度比 TUI 慢
* Windows 有 cmd 和 powershell 两种终端，而 powershell 有 旧界面的终端、[新界面的终端](https://github.com/Microsoft/Terminal)
  或者自己安装的 [alacritty](https://github.com/alacritty/alacritty) 之类的模拟器，从而每种终端界面使用相同的 nvim，体验都不一样，比如
  * 从 TUI 打开一个文件很麻烦，在资源管理器找到一个文件，又有至少两种方式从终端打开它：
    * 右键鼠标打开新的终端（通常你不会想要很多终端窗口）
    * 复制文件路径到已有的一个终端
      * 如果你的文件放在资源管理器左侧的快捷文件夹，又得右键打开所在目录（或者查看属性）去复制路径：
        因为地址栏对快捷文件夹不显示完整的源路径！

此外，对于一个最基本的实际使用需求：
* 在服务器上使用 nvim，你基本上只用一种 SSH 客户端连接，从而从 nvim 外复制粘贴到 nvim 内、从
  nvim 内复制粘贴到 nvim 外都能很快习惯；即便不同的 SSH 客户端，基本都有类似的设计
* 而在 Windows 上，你 **必须** 选好一种方式去习惯，每种终端界面都有自己的快捷键怪癖！对于 nvim
  这种严重依赖快捷键的程序，更改一次快捷键习惯，就是一次自找麻烦：
  * 选择 alacritty 终端界面：使用 `Ctrl-Shift-C/V` 从 nvim 复制到系统粘贴板/从系统粘贴板粘贴到 nvim
  * 选择 nvim-qt：使用原始的 `+` 寄存器，你得习惯 `"+y`、`<c-r>+` 等方式做上面的事情
  * 默认没有直接选中文字就复制到系统粘贴板和直接右键就粘贴到 nvim 这样人性化的操作，你需要它们就得折腾
    —— 所有 SSH 客户端都会开箱即用地提供这个吧 :)
    * 新界面的终端有这个选项，而且体验还不错，打开 nvim 选中复制多行是正常的（在正确的地方换行，而不是填充空格）
      ![term](https://user-images.githubusercontent.com/25300418/230635565-0a9953cf-d365-4a67-9143-90f33f4ba7e5.png)
    * 而 alacritty 可以修改配置（`~\AppData\Roaming\alacritty\alacritty.yml`）：

      ```yaml
      selection:
        save_to_clipboard: true
      mouse_bindings:
        - { mouse: Right, mode: ~Vi, action: PasteSelection  }
      ```

      然而，这在复制多行时，应该直接换行的地方填充了空格，从而仍然需要 `+` 寄存器的方式正确复制多行文本。

# 跨平台配置需要统一

讲完初步的界面使用区别，再谈一个跨平台最常见的难题：路径字符串。

众所周知，unix 使用 `/` 作为路径分隔符，而 Windows 使用 `\\`。单一平台下工作时，你从来不用理会这种差异，从而
配置文件里面随便用 `/` 拼接路径，但一旦在 Windows 重用 unix 的 nvim 配置文件，则需要改写所有相关代码。

而遗憾的是， nvim 没有内置统一的路径拼接函数，从而每个插件都可能有不同的做法：

* 判断不同的平台（`vim.loop.os_uname().version:match 'Windows'`、`jit.os == 'Windows'`），然后 `gsub` 全部替换（`path:gsub('\\', '/')`）
* 使用插件/外部库：[`require 'plenary.path'`](https://github.com/nvim-lua/plenary.nvim)

```lua
local path = require 'plenary.path'
local p = path:new("c:\\a","b")
print(p)
print(p:exists())
print(p:joinpath("d"))
```

