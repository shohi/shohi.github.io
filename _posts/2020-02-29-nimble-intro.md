---
layout: post
title: nimble使用介绍
category: tech
guid: D8E077F6-56A5-456E-9CE8-73A2E18614EC
tags: [tech, nim]

---
{% include JB/setup %}

[nimble](https://github.com/nimlang/nimble)是`nimlang`的package管理工具，包括新package的创建、依赖管理、测试、构建及发布等，本文将简要介绍它的用法. (关于[nimlang](https://nim-lang.org)可以访问官网进行了解)



### 1. 创建新package

`nimble init [new-package-name]`

根据提示信息一步步确认就可以了, 会在当前目录下创建一个以`[new-package-name]`命名的文件夹. 其中一个重要文件就是以package名称命名的`.nimble`文件, `nimble`使用这个文件对package进行管理.

nimble文件是一个[NimScript格式](https://github.com/nim-lang/nimble#nimble-init), 在其中可以使用nim的语法和函数, NimScripts详细说明见其[文档](https://nim-lang.org/docs/nims.html)。

```terminal
# example
$> cd ~/tmp
$> nimble init hello

$> cd hello && tree -L 3
.
├── hello.nimble
├── src
│   ├── hello
│   │   └── submodule.nim
│   └── hello.nim
└── tests
    ├── config.nims
    └── test1.nim
```


### 2. 添加依赖

修改`.nimble`文件, 使用`requires`函数.

```nim
# Dependencies
requires "nim >= 1.0.6", "jester > 0.1 & <= 0.5"
```

### 3. 测试

```terminal
$> nimble test
```

这个命令是封装了`nim c -r tests/test*.nim`, 将执行`tests`目录下所有以`test`开头的nim文件.

在nim中测试文件就是一个普通的可执行文件, 主要工作由[unittest](https://nim-lang.org/docs/unittest.html)模块来完成.  如果要精确控制执行哪些测试, 可以直接使用`nim c -r`.

```terminal
$> nim c -r test [glob_pattern]
```

### 4. 构建

```terminal
$> nimble build [bin-module]
```

在package根目录下运行上述命令, 将根目录下创建以相应module对应的可执行文件. nimble文件中指定要构建可执行文件的module, 即设置`bin`变量.

```nim
srcDir = "src"
bin    = @["hello"]
```

### 问题

1. nimble目前不支持通过命令行添加/删除依赖, 如`golang`中的`go add`.
2. nimble无法指定测试模式-仅运行匹配的测试, 如`go test -run [pattern]`及`cargo test [pattern]`.
3. nimble如何指定可执行文件的名称? 区别于module名称?
4. nimble只能构建二进制? 能否构建静态/动态库?
5. unittest仅支持glob匹配, 不支持包含匹配, 比如仅执行名称中包含"hello"的测试, 目前无法做到?


### Note

1. nimble文档, <https://github.com/nim-lang/nimble>
2. 文中所用`nim v1.0.6 (MacOSX)`及`nimble 0.11.0`.
