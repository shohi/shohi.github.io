---
layout: post
title: Vim中UltiSnips使用介绍
category: tech
guid: 81BEFA22-19E2-4E40-BBBD-27C073AB5EE6
tags: [tech, vim]

---
{% include JB/setup %}

`Snippet`是预定义的代码片段, 其与特定的触发条件绑定之后. 只要特定条件被触发，预定的代码片段将自动展开并插入到指定位置. `Snippet` 以一种编程的方式极大地减少了按键次数及出错的可能，能有效地提高编辑效率. 本文简要介绍如何结合`UltiSnips`自定义`Snippet`，而且同时兼容已有第三方的`Snippet`(如[honza/vim-snippets](https://github.com/honza/vim-snippets)).


## 1. UltiSnips

`UltiSnips`是`Vim`中比较流行的`Snippet`引擎，其主要功能是定义`Snippet`的语法(如何识别`Snippet`)、查找`Snippet`以及处理`Snippet`. `UltiSnips`负责处理`Snippet`,但其本身并不包含一些常用的`Snippet`. Vim中另外一个Plugin - [honza/vim-snippets](https://github.com/honza/vim-snippets), 则收集了大量的常用`Snippet`, 几乎涵盖所有的编程语言.

用[vim-plug](https://github.com/junegunn/vim-plug)可以很方便的安装这两个插件.

```vim
" Put following line between 
" call plug#begin('~/.vim/plugged')

Plug 'sirver/ultisnips'
Plug 'honza/vim-snippets'

" call plug#end()
```

### 1.1 Snippet语法
定义`Snippet`主要是指明其触发条件及对应的代码片段, 基本形式如下:

```
snippet KEYWORD "some description" <options>

[snippet body...]

endsnippet
```

其中触发条件是由`KEYWORD`和`options`而定, 一般情况下当输入`KEYWORD`后，按下`<TAB>`即触发了对应的`Snippet`处理. 确切地说，`Snippet`的触发条件是`KEYWORD`+`options`+ 启动UlitSnip处理程序的按键，通常触发`UltiSnip`处理程序的按键为`<TAB>`, 其可以通过`g:UltiSnipsExpandTrigger`变量设置, 如.

```vim
" Trigger configuration. Do not use <tab> if you use https://github.com/Valloric/YouCompleteMe.
let g:UltiSnipsExpandTrigger="<tab>"
```

关于`Snippet`语法及`options`设置等更为详细的介绍，可以参考[官方文档](https://github.com/SirVer/ultisnips/blob/master/doc/UltiSnips.txt), 或者在Vim中查看帮助文档.

```vim
:help UltiSnips
:help UltiSnips-basic-syntax
```

### 1.2 搜索路径
当特定条件被触发后, `UltiSnips`就会根据`KEYWORD`及当前所编辑的文件类型(`ft`)到`runtimepath`下指定的子目录中寻找`Snippet`的定义文件 - `<ft>.snippets` ，将`snippet body`取出并将其展开并插入到`KEYWORD`起始位置.

其中要搜索的子目录由以下变量指定, 默认是`UltiSnips`.

```vim
let g:UltiSnipsSnippetDirectories=["UltiSnips"]
```

`UltiSnips`会顺序遍历`runtimepath`下所有目录，如果其有子目录`UltiSnips`, 其将检查这个字母下是否有`<ft>.snippets`. 如果有进而检查这个文件上，是否有`KEYWORD` + `options`对应的条目，找到之后取出`snippet body`, 然后跳过该文件继续搜索下一个文件.(`runtimepath`一般是`~/.vim`, 至于是深度优先还是广度优先，可以使用一个自定义的`Snippet`进行测试).

**UltiSnips会先把所有符合的Snippet都找到, 然后按照选择最高优先级的`Snippet`进行展开处理**. `Snippet`的优先级是以文件为单位的，一个文件内的所有`Snippet`都共享一个优先级，通常优先级是`0`, 但是一些第三方定义的`Snippet`为了避免可能与用户自定义的有冲突, 会将优先级设置成负数或者很小的值，如`honza/vim-snippets`中[go.snippets](https://github.com/honza/vim-snippets/blob/62f46770378ab899f40c334de264ccd64dc2db57/UltiSnips/go.snippets#L3)的优先级是`-50`.


需要注意的是如果`UltiSnipsSnippetDirectories`指定了多个子目录, 那么它们是按顺序访问的. 也就说是首先把`runtimepath`遍历一遍查是否有第一个子目录, 如果没有，在把`runtimepath`遍历一次检查是否有第二个子目录.

另外`SnippetDirectories`也可以指定绝对路径, 具体信息可以查看帮助文档.

```vim
:help UltiSnips-how-snippets-are-loaded
```

## 2. Customized Snippet

由于默认情况下`UltiSnips`只会检查`runtimepath`下指定的子目录, 因此通常会将自定义的Snippet放在有别于`UltiSnips`的目录下，并将其链接到`~/.vim/`, 如

```bash
# download snippet-repo 
git clone <my/snippet-repo> <path/to/snippet-repo>

# create a symbolic link to snippet-repo
ln -s <path/to/snippet-repo> ~/.vim/MyUlitSnips

# add `MyUltiSnips` to SnippetDirectories
# let g:UltiSnipsSnippetDirectories=["MyUltiSnips", "UltiSnips"]
```

如果想让自定义的`Snippet`优先被选择，可以将对应的目录名放在第一个，反之，放在最后. 


### 2.1 Go Snippet Example

```
# go.snippets

snippet ptd "Panic With TODO" b
panic("TODO")
endsnippet
```
在编辑`golang`文件时, 在一行起始位置输入`ptd<TAB>`, 将自动转换为`panic("TODO")`.


### 2.2 Markdown Snippet Example


```
# markdown.snippets

snippet head "Jekyll post header" b

---
layout: post
title: ${1:title}
category: ${2:tech}
guid: `uuidgen`
tags: [${3:tech}]
---
{% include JB/setup %}
{$0}

endsnippet
```

这个`Snippet`可以用来快速插入Jekyll Post的Header部分, 其中`guid`的值由`shell`命令`uuidgen`产生. 其中`${}`表示占位符, `${0}`代表`Snippet`展开后`<TAB>`最终停留的位置.展开后, 可以通过快捷键来回切换到占位符的位置.

```vim
let g:UltiSnipsJumpForwardTrigger="<c-j>"
let g:UltiSnipsJumpBackwardTrigger="<c-k>"
```

`UltiSnips`除了支持占位符/`Shell`命令插值, 还有很多高级用法，详情请移步[官方文档](https://github.com/SirVer/ultisnips/blob/master/doc/UltiSnips.txt).

## 3. 参考链接

- <https://mednoter.com/UltiSnips.html>
- <https://jdhao.github.io/2019/04/17/neovim_snippet_s1/>
- <https://github.com/sirver/ultisnips>
