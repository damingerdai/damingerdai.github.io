---
title: noice.nvim在cmdline中的光标自动滚动到最前面的bug
date: 2025-05-22 21:58:27
tags: [neovim, iterm2, nocie.nvim]
categories: [软件]
---

# 前言

最新把 neovim 升级到最新版本[0.11.1](https://github.com/neovim/neovim/releases/tag/v0.11.1),lazyvim 升级到[14.x](https://www.lazyvim.org/news#14x)。然后不出意外就挂了好几个 plugin。其中 noice.nvim 是影响比较小的但比较膈应人的。

# bug

## 描述

{% raw %}
<video src="https://private-user-images.githubusercontent.com/16384908/443705864-b981f9d2-17c7-452b-8fae-916b5a2dfbf2.mov?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDc5MjM0MzEsIm5iZiI6MTc0NzkyMzEzMSwicGF0aCI6Ii8xNjM4NDkwOC80NDM3MDU4NjQtYjk4MWY5ZDItMTdjNy00NTJiLThmYWUtOTE2YjVhMmRmYmYyLm1vdj9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTA1MjIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwNTIyVDE0MTIxMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQwZTk2YmRmOTY4MDRhYjk0MDNmMzgyMzI3YjliZGNkMTQ1NmUwYWE0NDc0YTBjODA0MzMwY2ZlYjRjNWNjYTkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.lhQ_rFTJKjP7UErvMKu2B4vLRQpXY7j3hYhzJ6AjWfw" data-canonical-src="https://private-user-images.githubusercontent.com/16384908/443705864-b981f9d2-17c7-452b-8fae-916b5a2dfbf2.mov?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDc5MjM0MzEsIm5iZiI6MTc0NzkyMzEzMSwicGF0aCI6Ii8xNjM4NDkwOC80NDM3MDU4NjQtYjk4MWY5ZDItMTdjNy00NTJiLThmYWUtOTE2YjVhMmRmYmYyLm1vdj9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTA1MjIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwNTIyVDE0MTIxMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQwZTk2YmRmOTY4MDRhYjk0MDNmMzgyMzI3YjliZGNkMTQ1NmUwYWE0NDc0YTBjODA0MzMwY2ZlYjRjNWNjYTkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.lhQ_rFTJKjP7UErvMKu2B4vLRQpXY7j3hYhzJ6AjWfw" controls="controls" muted="muted" class="d-block rounded-bottom-2 border-top width-fit" style="max-height:640px; min-height: 200px" ></video>
{% endraw%}

当在cmdline模式输入命令的时候光标总是会莫名其妙的滚动到最前面。我本来以为是noice.nvim的问题，但是经过测试，发现只有在itermn2的非全屏模式下才会出现，全屏模式或者其他终端模拟器则不会出现类似的问题。

## 解决

好在[bug: Cursor jumps in cmdline - 4.5.0](https://github.com/folke/noice.nvim/issues/923#issuecomment-2898259739)中有小哥也遇到过类似的问题，他在iTerm2中禁用"由会话触发的窗口调整大小"功能解决了这个问题（设置路径：Settings > Profiles > [我的配置文件] > Terminal > 勾选"Disable session-initiated window resizing"），然后就好了。

## 其他

在[https://superuser.com/questions/113944/how-to-prevent-screen-from-resizing-my-terminal-in-mac-os-x](https://superuser.com/questions/113944/how-to-prevent-screen-from-resizing-my-terminal-in-mac-os-x), 看上去iterm2的这个bug存在挺久的了。