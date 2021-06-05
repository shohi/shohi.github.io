---
layout: post
title: golang之exec
category: tech
guid: 88E7BC20-3B0F-4C02-BF1D-94074F91B2A5
tags: [tech]

---
{% include JB/setup %}

`exec.Command`用于启动一个子进程来执行指定的命令, 它是golang中调用外部命令的常用方式. `exec.Command`使用大概分成以下几步

1. 创建`exec.Command`数据结构
2. 启动执行--调用`Start`方法
3. 等待执行结束--调用`Wait`方法

其中`Start`并不等待命令执行完才返回, 只要启动了就返回. 当`Start`返回后, `Command`中最重要的成员变量`Process`就被设置了. 通过`Process`这个成员, 可以得到子进程的ID--`Process.Pid`, 以及对子进程进行控制-- 如发送信号. 而其中最常用的是终止子进程--`Process.Kill()`.

关于`Process.Kill()`, 有几点需要注意.

## 1. 派生进程

`Cmd`的默认配置下, 只终止该子进程,如果该子进行又派生了其他进行, 那么这些派生进行默认是不会被终止的. 当子进程终止后, 它们变成了Orphan进程. 但是`Command`有一个选项可以让子进程关闭后由其派生的进程, **这个选项应该始终被设置**. (具体细节可参考Link 1/2/3)

```go

	cmd := exec.Command("/bin/sh", "-c", "watch date > date.txt")
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}

	start := time.Now()
	time.AfterFunc(3*time.Second, func() {
		cmd.Process.Kill()
	})
	err := cmd.Run()

	fmt.Printf("pid=%d duration=%s err=%s\n",
		cmd.Process.Pid, time.Since(start), err)
```

## 2. 监听Socket
如果子进程中启动了Socket监听了, `Process.Kill`之后再立即重启, client感受不到这种变化(??).






## Link

1. 结束子进程以及它的子进程, <https://studygolang.com/articles/10083>
2. Terminating Processes in Go, <https://bigkevmcd.github.io/go/pgrp/context/2019/02/19/terminating-processes-in-go.html>
