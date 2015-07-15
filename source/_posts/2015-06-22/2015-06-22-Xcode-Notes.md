title: Xcode Notes
categories: Xcode
tags:
  - IOS
description:
date: 2015-06-22 16:57:42
author:
photos:
---
### 快捷键
```
⌘（command）、⌥（option）、⇧（shift）、⇪（caps lock）、
⌃（control）、↩（return）、⌅（enter）

^ + 1                 corresponding Nib/Storyboard file
^ + 2                 show the previous history
^ + 6                 methods list

^ + ⌘ + left         go back
⌘ + Shift + O        jump to a particular filename method
⌘ + L                go to line
⌘ + Y                toggle all breakpoints
⌘ + 0                toggle left panel
⌘ + 1                show Project Nacigator

^ + P                 移动光标到上一行
^ + N                 移动光标到下一行
^ + A                 移动光标到本行行首
^ + E                 移动光标到本行行尾
^ + D                 删除光标右边的字符
^ + K                 删除空行/光标到行尾
^ + L                 将插入点置于窗口正中

⌘ + ⌥ + d           显示／隐藏 dock

Fn + Delete           删除光标后的一个字符
⌘ + Delete           删除光标至行首的内容

⌘ + ⇧ + ;           调出拼写检查对话框。
⌘ + ⌥ + =           xib update frame
```
<!-- more -->

#### 全局删除光标右边字符
```
^ + D                 删除光标右边的字符
⌘ + ^ + D            直接调用字典
```

### xcode target build setting
```
    XCBuildConfiguration
```


#### xcode plugin
```
~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins
~/Library/Developer/CoreSimulator
```

#### Xcode plugins
```
FuzzyAutocomplete
KSImageNamed
HOStringSense
VVDocumenter-xcode
```

### xcode 环境变量 Environment variable
```
OBJC_PRINT_LOAD_METHODS
OBJC_PRINT_REPLACED_METHODS
OBJC_PRINT_DEPRECATION_WARNINGS
OBJC_DEBUG_NIL_SYNC
NSZombieEnabled     !!
XcodeColors
```

### xcode 7 Enable Address Sanitizer
![](http://cc.cocimg.com/api/uploads/20150616/1434421492186513.png)
打开的方法: Schema->Run->Diagnostics  Enable Address Sanitizer


### delevoper tools
```
colorsnapper
prepo
simpholders
xscope
kaleidoscope
```

### xcode 模拟器地址
    /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/
    https://developer.apple.com/downloads/index.action

### 开发工具
```
xcode-select --install
```

### xocde uuid
```
defaults read /Applications/Xcode-beta.app/Contents/Info DVTPlugInCompatibilityUUID
```



