---
layout: post
title: vscode之python环境设置
category: tech
guid: 0ee5a216-65a4-4159-bd96-e1fa12fdb561
tags: [Python, vscode]

---
{% include JB/setup %}

使用`vscode`设置`python`开发环境，主要有以下方面:

`1.` 语法高亮显示，代码自动补及支持debug等  
`2.` 设置`python`路径及`virtualenv`  
`3.` 易于执行代码

这三个方面涉及到两个插件以及一个vscode配置.

### 1. `Python`插件
安装`Python`插件以支持语法高亮、拼写检查及debug等

### 2. `vscode`设置
设置`vscode`内置的`python`路径--即`virtualenv`，以便于vscode安装一些其他的module以及调试:

```bash
Shift + Command + P
// type 
Python: Select Interpreter
```

上述步骤也可以通过手工设置方式完成, `Command + ,`编辑`User Setttings`, 添加如下代码:


```
"python.pythonPath": "${workspaceFolder}/venv/bin/python3"
``` 

### 3. `Code Runner`插件
安装`Code Runner`插件，以便于执行Python脚本. 但是需要注意的`Code Runner`使用其自身指定的`python`路径, 需要更改为第2步中设置的`virtualenv`. **由于Code Runner内部所用变量不一定与vscode相同，因此要依据其doc上的说明!**

编辑`User Setting`, 添加如下代码

```json
"code-runner.executorMap": { 
	"python": "$workspaceRoot/venv/bin/python $fullFileName" 
} 
```

### 4. 参考链接
`1.` 用VSCode写python的正确姿势, <http://www.cnblogs.com/bloglkl/archive/2016/08/23/5797805.html>  
`2.` Visual Studio code and virtualenv,<https://stackoverflow.com/questions/38545326/visual-studio-code-and-virtualenv>  
`3.` Code Runner for Visual Studio Code, <https://github.com/formulahendry/vscode-code-runner>  

*注: 所有操作在macOS进行, 其他系统会有所差别*


*The End.*
