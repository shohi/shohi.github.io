---
layout: post
title: golang之signal
category: tech
guid: 1989251F-A847-4A98-B0B4-EF31886A085A
tags: [tech, golang, os]

---
{% include JB/setup %}

Golang中可以使用如下方式捕获Signal, 如`SIGTERM`/`SIGUSER`, 收到这个信号后, 程序就有能力做一些定制化的操作, 比如`Hot Load`/`Graceful Shutdown`等.

```go
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh,
		syscall.SIGINT,
		syscall.SIGTERM,
	)

    for sig := range sigCh {
        log.Printf("receive signal: %vn", sig)
        // do something
    }
```

但是有两个信号需要特别注意--`SIGKILL`和`SIGSTOP`, 它们不一定能被捕获.

```
The signals SIGKILL and SIGSTOP may not be caught by a program, and therefore cannot be affected by this package.
```

## Links

1. Package signal, <https://golang.org/pkg/os/signal/>
1. Linux Signal及Golang中的信号处理, <https://colobu.com/2015/10/09/Linux-Signals/>
