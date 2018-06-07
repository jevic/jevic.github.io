---
layout: post
categories: Tools
title: sublimeText3 插件配置快捷操作
date: 2017-12-29 13:07:51 +0800
description: sublimeText3
keywords: sublimetext ide
---

## SublimeText3 for Mac OS
> 工欲善其事必先利其器 


## 禁止自动更新
![](http://ourpypwzb.bkt.clouddn.com/sublimeText_User.png)
1. 添加 "update_check": false,
2. 点击菜单栏 Help --> license 输入以下注册信息

``` sh
—– BEGIN LICENSE —–
Michael Barnes
Single User License
EA7E-821385
8A353C41 872A0D5C DF9B2950 AFF6F667
C458EA6D 8EA3C286 98D1D650 131A97AB
AA919AEC EF20E143 B361B1E7 4C8B7F04
B085E65E 2F5F5360 8489D422 FB8FC1AA
93F6323C FD7F7544 3F39C318 D95E6480
FCCC7561 8A4A1741 68FA4223 ADCEDE07
200C25BE DBBC4855 C4CFB774 C5EC138C
0FEC1CEF D9DCECEC D3A5DAD1 01316C36
—— END LICENSE ——
```

## 安装 Package Control
> 所有插件都需要使用此工具来进行安装! 

1. 打开 Sublime Text 3
2. 点击顶部菜单的 View -> Show Console
3. 输入以下代码或者直接访问此链接获取 https://packagecontrol.io/installation#st3

```
import urllib.request,os,hashlib; h = 'df21e130d211cfc94d9b0905775a7c0f' + '1e3d39e33b79698005270310898eea76'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```
![](http://ourpypwzb.bkt.clouddn.com/package01.png)

- 点击回车 安装完后即可通过 `Command + Shift + P` 打开 Package Control 来安装插件

![](http://ourpypwzb.bkt.clouddn.com/package02.png)

##

## 插件
### 推荐插件 
1. Emmet 概括的说，Emmet(其前身是Zen Coding) 是一个可以让你更快更高效地编写HTML/CSS，可以节省你大量时间的插件。 
调用Emmet快捷键：⌃⌥↩。 
2. Git 这个插件会将git整合进你的SublimeText，使的你可以在SublimeText中运行Git命令，包括添加，提交文件，查看日志，文件注解以及其它Git功能。 
3. AutoFileName 自动补全文件路径，非常方便。 
4. DocBlockr DocBlockr会成为你编写代码文档的有效工具。当输入/**并且按下Tab键的时候，这个插件会自动解析任何一个函数并且为你准备好合适的模板 
5. SFTP 快速编辑远程服务器文件 
6. SublimeLinter 行内语法检测插件，支持： C/C++, Java, Python, PHP, js, HTML, CSS, etc. 
7. Alignment 简单到极致的多行选择和多行选择对齐插件 
8. Markdown-preview Markdown 
9. ChineseLocalization Sublime 汉化插件
10. Anaconda Python
11. ctags
### view HTML插件
- shift + command + p 
- 输入pcip 选择Install Package并回车
- view 回车安装

```
[  
{ "keys": ["f2"], "command": "open_in_browser" },
]

```

![](http://ourpypwzb.bkt.clouddn.com/viewhtml_01.png)
![](http://ourpypwzb.bkt.clouddn.com/viewhtml_02.png)
![](http://ourpypwzb.bkt.clouddn.com/viewhtml_set_03.png)
![](http://ourpypwzb.bkt.clouddn.com/viewhtml_key_04.png)

### 浏览markdown文件
- Command + Shift + P
- pcip
- mp

### Python 必备
- anaconda

``` sh
{
    "color_scheme": "Packages/Color Scheme - Default/Monokai.tmTheme",
    "font_size": 14,
    "ignored_packages":
    [
        "Vintage"
    ],
    "anaconda_linting": false ## 取消Python 编写时白色框框提示
}
``` 


## SublimeText3 for Mac OS 常用快捷键
### 符号说明

⌘：command 
⌃：control 
⌥：option 
⇧：shift 
↩：enter 
⌫：delete 
打开/关闭/前往

#### 快捷键 功能 
⌘⇧N 打开一个新的sublime窗口 
⌘N 新建文件 
⌘⇧W 关闭sublime，关闭所有文件 
⌘W 关闭当前文件 
⌘P 跳转、前往文件、前往项目、命令提示、前往method等等（Goto anything） 
⌘⇧T 重新打开最近关闭的文件 
⌘T 前往文件 
⌘⌃P 前往项目 
⌘R 前往method 
⌘⇧P 命令提示 
⌃G 前往行 
⌘KB 开关侧栏 
⌃` 打开控制台 
⌃- 光标跳回上一个位置 
⌃⇧- 光标恢复位置 
#### 快捷键 编辑
 
⌘A 全选 
⌘L 选择行（重复按下将下一行加入选择） 
⌘D 选择词（重复按下时多重选择相同的词进行多重编辑） 
⌃⇧M 选择括号的内容 
⌘⇧↩ 在当前行前插入新行 
⌘↩ 在当前行后插入新行 
⌃⇧K 删除行 
⌘KK 从光标处删除至行尾 
⌘K⌫ 从光标处删除至行首 
⌘⇧D 复制（多）行 
⌘J 合并（多）行 
⌘KU 改为大写 
⌘KL 改为小写 
⌘C 复制 
⌘X 剪切 
⌘V 粘贴 
⌘/ 注释 
⌘⌥/ 块注释 
⌘Z 撤销 
⌘Y 恢复撤销 
⌘⇧V 粘贴并自动缩进 
⌘⌥V 从历史中选择粘贴 
⌃M 跳转至对应的括号 
⌘U 软撤销（可撤销光标移动） 
⌘⇧U 软重做（可重做光标移动） 
⌘⇧S 保存所有文件 
⌘] 向右缩进 
⌘[ 向左缩进 
⌘⌥T 特殊符号集 
⌘⇧L 将选区转换成多个单行选区 

#### 查找/替换
⌘f 查找 
⌘⌥f 查找并替换 
⌘⌥g 查找下一个符合当前所选的内容 
⌘⌃g 查找所有符合当前选择的内容进行多重编辑 
⌘⇧F 在所有打开的文件中进行查找 
#### 拆分窗口/标签页


⌘⌥[1,2,3,4] 单列、双列、三列、四列 
⌘⌥5 网格（4组） 
⌃[1,2,3,4] 焦点移动到相应的组（分屏编号） 
⌃⇧[1,2,3,4] 将当前文件移动到相应的组（分屏编号） 
⌘[1,2,3,4] 选择相应的标签页 
#### 快捷操作

⌘⌃上下键 两行交换位置 
⌘KB 显示/隐藏侧边


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
