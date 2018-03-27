---
title: Mac下vim开启语法高亮&着色
date: 2014-01-20 12:06:34
categories: Mac
tags:
    - Linux
    - vim
    - Mac
---


Mac OS并不像大多数Linux发行版vim默认自带语法着色高亮显示（通常Linux可通过编辑/etc/vimrc进行全局设置或~/vimrc进行单用户设置），使用vi/vim编辑文件时很不方便，如何解决 ?


# 编辑文件/usr/share/vim/vimrc
```bash
BobZhao@mac:~ > sudo vim /usr/share/vim/vimrc
Password:
" Configuration file for vim
set modelines=0         " CVE-2007-2438

" Normally we use vim-extensions. If you want true vi-compatibility
" remove change the following statements
set nocompatible        " Use Vim defaults instead of 100% vi compatibility
set backspace=2         " more powerful backspacing

" Don't write backup file if vim is being called by "crontab -e"
au BufWrite /private/tmp/crontab.* set nowritebackup
" Don't write backup file if vim is being called by "chpass"
au BufWrite /private/etc/pw.* set nowritebackup
```

<!-- more -->

# 在set backspace=2下插入配置
```bash
set ai                  " auto indenting
set history=100         " keep 100 lines of history
set ruler               " show the cursor position
syntax on               " syntax highlighting
set hlsearch            " highlight the last searched term
filetype plugin on      " use the file type plugins

" When editing a file, always jump to the last cursor position
autocmd BufReadPost *
\ if ! exists("g:leave_my_cursor_position_alone") |
\ if line("'\"") > 0 && line ("'\"") <= line("$") |
\ exe "normal g'\"" |
\ endif |
\ endif
```

# 验证
再次打开该文件发现已经生效
![configure-vim](/images/post/2014/10/01/configure-vim-highlight-on-mac.png)


