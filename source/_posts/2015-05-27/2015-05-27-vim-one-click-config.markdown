---
layout: post
title: "一键配置vim"
date: 2015-05-27 15:54:23 +0800
comments: true
categories: Vim
---
### 一键配置vim
```
curl https://raw.githubusercontent.com/spf13/spf13-vim/3.0/bootstrap.sh -L > spf13-vim.sh && sh spf13-vim.sh
```

### bug更正
    
```
sublime vundle-vimfiles/plugin/settings/CtrlP.vim
```


----------
before `let g:ctrlp_custom_ignore = ..` insert below code

```
unlet g:ctrlp_custom_ignore
```

参考自: https://github.com/spf13/spf13-vim/issues/444
    