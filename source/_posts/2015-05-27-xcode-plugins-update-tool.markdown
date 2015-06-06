---
layout: post
title: "更新XcodePlugins脚本"
date: 2015-05-27 15:55:07 +0800
comments: true
categories: Xcode
---
ios8.2  ------ A16FF353-8441-459E-A50C-B071F53F51B7
ios8.3 ------- 9F75337B-21B4-4ADC-B558-F9CADF7073A7

```
#!/bin/sh

PLIST_BUDDY=/usr/libexec/PlistBuddy

function add_compatibility() {
  "$PLIST_BUDDY" -c "Add DVTPlugInCompatibilityUUIDs:10 string $2" \
    "$1/Contents/Info.plist"

}

function has_compatibility() {
  $PLIST_BUDDY -c 'Print DVTPlugInCompatibilityUUIDs' \
    "$1/Contents/Info.plist"|grep -q "$2"
  return $?

}

cd "$HOME/Library/Application Support/Developer/Shared/Xcode/Plug-ins"

for file in `ls -d *`
do

  if `has_compatibility "$file" 9F75337B-21B4-4ADC-B558-F9CADF7073A7`
  then
    true
  else
    echo "Plugin $file is now compatible with Xcode 6.0 GM"
    add_compatibility "$file" 9F75337B-21B4-4ADC-B558-F9CADF7073A7
  fi
done
```