---
title: "使用Done Channel或Context取消GoRoutine区别"
layout: post
date: 2020-01-12 23:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- markdown
- elements
star: true
category: blog
author: welly
description: context、channel、routine
---

## Abstract
本文使用两个携程去执行greeting和farewell，并使用Done Channel和context的模式去取消携程，对比两种方式的不同以及各自的优势。

## Done Channel
Done Channel模式将初始化done channel变量并传给每个携程来设置标准的抢占方法。使用下面代码中1方式，那么程序将会正常结束；使用2方式，我们在两个携程结束之前关闭done channel，那么两个携程都将会被提前取消。

```golang
package main

import (
    `fmt`
    `sync`
    `time`
)

func main() {
    var wg sync.WaitGroup

    done := make(chan interface{})
    defer close(done) // 1
    wg.Add(1)

    go func() {
        defer wg.Done()
        if err := printGreeting(done); err != nil {
            fmt.Printf("%v\n", err)
            return
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewell(done); err != nil {
            fmt.Printf("%v\n", err)
            return
        }
    }()
    //close(done) // 2
    wg.Wait()

}

func printGreeting(done <-chan interface{}) error {
    greeting, err := genGreeting(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewell(done <-chan interface{}) error {
    farewell, err := genFarewell(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}


func genGreeting(done <-chan interface{}) (string, error) {
    switch l, err := locale(done); {
    case err != nil:
        return "", err
    case l == "awesome":
        return "hello", nil

    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewell(done <-chan interface{}) (string, error) {
    switch l, err := locale(done); {
    case err != nil:
        return "", err
    case l == "awesome":
        return "goodbye", nil

    }
    return "", fmt.Errorf("unsupported locale")
}

func locale(done <-chan interface{}) (string, error) {
    select {
    case <-done:
        return "", fmt.Errorf("canceled")
    case <-time.After(3 * time.Second):
    }
    return "awesome", nil
}
```

``` text
运行1代码结果
goodbye world!
hello world!
```

``` text
运行2代码结果
canceled
canceled
```

## Context
如果我们希望genGreeting在耗时过长的时候发生超时；亦或是知道genFarewell的父进程将要被取消，不希望Farewell调用locale。那么context包中WithCancel、WithDeadline、WithTimeOut可以帮我们做这些事情。

```golang
package main

import (
    `context`
    `fmt`
    `sync`
    `time`
)

func main() {
    var wg sync.WaitGroup

    // 1 使用context包中的withCancel创建一个ctx
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    wg.Add(1)

    go func() {
        defer wg.Done()
        // 2 如果greeting发生错误那么ctx将会被取消
        if err := printGreetingCtx(ctx); err != nil {
            fmt.Printf("greeting: %v\n", err)
            cancel()
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewellCtx(ctx); err != nil {
            fmt.Printf("farewell: %v\n", err)
            cancel()
        }
    }()

    wg.Wait()

}

func printGreetingCtx(ctx context.Context) error {
    greeting, err := genGreetingCtx(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewellCtx(ctx context.Context) error {
    farewell, err := genFarewellCtx(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}


func genGreetingCtx(ctx context.Context) (string, error) {
    // 3 使用context包中withTime包装成新的ctx 如果该函数在1s中没执行完
    //   那么它的上游函数的任何子函数将会被取消
    ctx,cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()
    switch l, err := localeCtx(ctx); {
    case err != nil:
        return "", err
    case l == "awesome":
        return "hello", nil

    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewellCtx(ctx context.Context) (string, error) {
    switch l, err := localeCtx(ctx); {
    case err != nil:
        return "", err
    case l == "awesome":
        return "goodbye", nil

    }
    return "", fmt.Errorf("unsupported locale")
}

func localeCtx(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        // 4 返回ctx被取消的原因
        return "", ctx.Err()
    case <-time.After(3 * time.Second):
    }
    return "awesome", nil
}
```
```text
运行代码结果
greeting: context deadline exceeded
farewell: context canceled
```
## Conclusion
Done Channel模式使用抢占的方式让我们在父进程中取消所有的携程；而使用context可以提供了更多的功能让我们选择何时结束携程，而且我们可以从context错误返回中获取错误和超时的额外信息，这是Done Channel无法做到的。
































