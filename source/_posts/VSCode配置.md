---
title: VSCode配置
date: 2018-05-05 19:30:34
tags:
    - 前端
---

最近使用IDE从WebStorm更换到VSCode，故在此记录下VSCode的一些配置，供以后查看。

## 插件篇

VSCode更倾向于定制化，所以需要自行寻找需要的插件，来提高打码的效率~

我这里使用了一些暂时需要用到的插件：

<!-- more -->

### 1. Beautify
一个很常用的代码格式化工具，一键格式化，非常方便。

### 2. Git History

查看log，文件历史记录等等，在自带git功能上多加了许多功能。

### 3. Vetur

Vue的火热使得前端很多时候都使用它来开发，该插件高亮vue代码以及vue的代码格式化。

### 4. Auto Rename Tag

自动重命名tag名称，类似于webstorm中的功能。

### 5. Auto Close Tag

自动关闭tag，类似webstorm。

### 6. CSS Peek

自动识别css文件中的类名，在模板中插入class时自动提示。

## 配置篇

记录自己用的：

```js
{
    "git.enableSmartCommit": true,
    "git.autofetch": true,
    "terminal.integrated.shell.osx": "zsh",
    "window.zoomLevel": 0,
    "beautify.language": {
        "js": {
            "type": [
                "javascript",
                "json"
            ],
            "filename": [
                ".jshintrc",
                ".jsbeautifyrc"
            ]
        },
        "css": [
            "css",
            "scss"
        ],
        "html": [
            "htm",
            "html",
            "vue"
        ]
    },
    "html.format.endWithNewline": true,
    "prettier.singleQuote": true,
    "prettier.semi": false,
    "vetur.format.defaultFormatter.html": "js-beautify-html",
    "vetur.format.defaultFormatterOptions": {
      "wrap_attributes": "force-aligned"
    }
}
```