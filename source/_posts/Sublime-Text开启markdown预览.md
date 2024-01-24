---
title: Sublime_Text开启markdown预览
date: 2023-07-25 15:05:08
tags:
---

# Sublime_Text开启markdown预览

目前 `Sublime Text` 原生支持 `Markdown` 语法高亮, 但是还是没法像 `typora` 一样直接把文字渲染成文档. 现在的解决方法是生成一个网页并打开.

## 安装 `Markdown Preview` 插件

如果没有安装 `Package Control` 则先安装 (`Ctrl + Shift + P` 输入 `install package control`, 过一会就会弹窗说安装完成), 然后`Ctrl + Shift + P` 输入 `install package`, 输入 `Markdown Preview`. 操作时其实不用完整输入因为会自动补全. 

## 开启预览

打开一个 `Markdown` 文件, 按下 `Ctrl + Shift + P` 输入 `preview`, 可以看到第一个匹配的应该就是 `Markdown Preview` 的 `Preview in Browser`. 理论上回车之后就会在浏览器中打开渲染好的界面, 如果要实时刷新的话也可以网上参考下 `LiveReload` 相关的资料, 总体来说这部分教程很多而且没有难度就不赘述了.

## 公式渲染

数学公式是非常重要的部分, 为了启用公式渲染我们需要加载 `Mathjax` 对应的 `JavaScript` 脚本. 设置 `mathjax` 支持需要在 `Preferences ->Package Settings->Markdown Preview->Setting User` 中输入如下代码

``` json
{
    "enable_mathjax": true,
    "js": [
        "https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js",
        "res://MarkdownPreview/js/math_config.js"
    ],
    "markdown_extensions": {
        "pymdownx.arithmatex": {
            "generic": true
        }
    },
}
```

这样公式就可以正常显示了, 但是又会发现语法高亮没了. 这是因为我们在设置 `markdown_extensions` 的时候相当于覆盖了原先的扩展, 所以其它扩展都不在了. 因此我们要把原先的 `markdown_extensions` 完整拷贝到 `User` 文件, 然后加入最后一句. 完整的配置文件如下:

``` JSON
{
    "enable_mathjax": true,
    "js": [
    "https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js",
            "res://MarkdownPreview/js/math_config.js",
    ],
    "markdown_extensions": [
        // Python Markdown Extra with SuperFences.
        // You can't include "extra" and "superfences"
        // as "fenced_code" can not be included with "superfences",
        // so we include the pieces separately.
        "markdown.extensions.footnotes",
        "markdown.extensions.attr_list",
        "markdown.extensions.def_list",
        "markdown.extensions.tables",
        "markdown.extensions.abbr",
        "pymdownx.betterem",
        {
            "markdown.extensions.codehilite": {
                "guess_lang": false
            }
        },
        // Extra's Markdown parsing in raw HTML cannot be
        // included by itself, but "pymdownx" exposes it so we can.
        "pymdownx.extrarawhtml",

        // More default Python Markdown extensions
        {
            "markdown.extensions.toc":
            {
                "permalink": "\ue157"
            }
        },
        "markdown.extensions.meta",
        "markdown.extensions.sane_lists",
        "markdown.extensions.smarty",
        "markdown.extensions.wikilinks",
        "markdown.extensions.admonition",

        // PyMdown extensions that help give a GitHub-ish feel
        {
            "pymdownx.superfences": { // Nested fences and UML support
                "custom_fences": [
                    {
                        "name": "flow",
                        "class": "uml-flowchart",
                        "format": {"!!python/name": "pymdownx.superfences.fence_code_format"}
                    },
                    {
                        "name": "sequence",
                        "class": "uml-sequence-diagram",
                        "format": {"!!python/name": "pymdownx.superfences.fence_code_format"}
                    }
                ]
            }
        },
        {
            "pymdownx.magiclink": {   // Auto linkify URLs and email addresses
                "repo_url_shortener": true,
                "repo_url_shorthand": true
            }
        },
        "pymdownx.tasklist",     // Task lists
        {
            "pymdownx.tilde": {  // Provide ~~delete~~
                "subscript": false
            }
        },
        {
            "pymdownx.emoji": {  // Provide GitHub's emojis
                "emoji_index": {"!!python/name": "pymdownx.emoji.gemoji"},
                "emoji_generator": {"!!python/name": "pymdownx.emoji.to_png"},
                "alt": "short",
                "options": {
                    "attributes": {
                        "align": "absmiddle",
                        "height": "20px",
                        "width": "20px"
                    },
                    "image_path": "https://assets-cdn.github.com/images/icons/emoji/unicode/",
                    "non_standard_image_path": "https://assets-cdn.github.com/images/icons/emoji/"
                }
            }
        },
        {
            "pymdownx.arithmatex": { 
                "generic": true 
            }
        }
    ],
}
```

## 解决 `MacOS` 下字体问题

在 `Windows` 下修改配置文件后公式应该就可以正常渲染了, 但是在 `MacOS` 下公式中的字体会显得特别奇怪. 查看 HTML 文件发现字体是 `STIX-Regular`, 好像也没什么问题. 网上对这个的讨论比较少, 所以我没能找到具体的原因, 但是可以通过更改公式渲染的方式解决字体异常的问题.

查看生成的 HTML 文件, 我们发现 `math_config.js` 文件设置输出的格式为 `HTML-CSS`. 但是奇怪的是我没能找到这个 `res://MarkdownPreview/js/math_config.js` 文件存放在哪儿, 如果有知道的可以邮件告知我下不胜感谢. 我的解决方法是直接在本地建立一个新的 `js` 文件来引用, 其中的内容就是 `math_config.js` 的内容但是修改第 `4` 行为 `jax: ["input/TeX", "output/CommonHTML"],` . 这里可选的选项为:

- "output/CommonHTML"
- "output/HTML-CSS"
- "output/NativeMML"
- "output/PlainSource"
- "output/PreviewHTML"
- "output/SVG"

使用 `CSS` 会出问题, 选择 `SVG` 会生成矢量图不能复制, 使用 `NativeMML` 复制会导致乱码. 此处的列表可以放入不止一个选项, 此时仍然正常显示但是具体区别还没有验证.

此外, 引用别人 `CDN` 的内容总是不太好, 尤其是在本地预览时如果没有网就加载不了了. 所以我们干脆把 `Mathjax` 文件放到本地, 从[这里](MathJax-2.7.9.tar.gz)或者 `Github` 下载源文件 (注意一定要`2.xx.xx` 版本) 后解压到合适的目录, 并修改配置文件中的 `JS` 文件地址如下:

``` JSON
{
	    "js": [
        "file://<PATH-TO-MATHJAX-FOLDER>/MathJax.js",
        "file://<PATH-TO-CONFIG-FILE>/math_config.js"
    ],
}
```

再次预览发现字体已经正常了.

