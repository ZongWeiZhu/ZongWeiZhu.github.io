---
title: "GoLang 循环调用闭包、携程的问题"
layout: post
date: 2019-12-22 22:44
image: /assets/images/markdown.jpg
headerImage: false
tag:
- markdown
- elements
star: true
category: blog
author: welly
description: closure、routine
---

## 循环调用闭包

GoLang的for循环，使用同一个迭代变量(地址相同)，当我们使用闭包时，如果使用的姿势不对会造成意想不到的结果，如下图这个例子，根本原因在于闭包延迟使用了这个迭代变量。

``` golang
func main() {
    var dummyFuncs []func()
    for i := 0; i < 5; i++ {
        dummyFuncs = append(dummyFuncs, func() {
            fmt.Println(i)
        })
    }
    for i := 0; i < 5; i++ {
        dummyFuncs[i]()
    }
}
```
``` text
运行结果：
5
5
5
5
5
```

### 闭包的两种方法解决

#### 1 使用新的局部变量
``` golang
func main() {
    var dummyFuncs []func()
    for i := 0; i < 5; i++ {
        tmp := i
        dummyFuncs = append(dummyFuncs, func() {
            fmt.Println(tmp)
        })
    }
    for i := 0; i < 5; i++ {
        dummyFuncs[i]()
    }
}
```
``` text
运行结果：
0
1
2
3
4
```
#### 2 使用新的函数将迭代变量作为参数传入
``` golang
func main() {
    var dummyFuncs []func()
    for i := 0; i < 5; i++ {
        func(j int){
            dummyFuncs = append(dummyFuncs, func() {
                fmt.Println(j)
            })
        }(i)
    }
    for i := 0; i < 5; i++ {
        dummyFuncs[i]()
    }
}
```
``` text
运行结果：
0
1
2
3
4
```

## 循环调用携程
当启动10000个携程时，期望获得从1-10000的输出，但是从输出文件中发现重复的数字非常多。因此，延迟使用循环里的迭代变量也会导致携程出现同样的问题。
``` golang
func main(){
    wg := sync.WaitGroup{}
    wg.Add(10000)
    for i:=0 ; i<10000 ; i++ {
        go func() {
            defer  wg.Done()
            fmt.Println(i)
        }()
    }
    wg.Wait()
}
```
``` bash
go run share_mem.go > share_mem | sort -u > share_mem_2
```
``` text
运行结果：
990
991
992
996
997
999
```

### 携程的两种方法解决

#### 1 使用新的局部变量
``` golang
func main(){
    wg := sync.WaitGroup{}
    wg.Add(10000)
    for i:=0 ; i<10000 ; i++ {
        j := i
        go func() {
            defer  wg.Done()
            fmt.Println(j)
        }()
    }
    wg.Wait()
}
```
``` bash
go run share_mem.go > share_mem | sort -u > share_mem_2
```
``` text
运行结果：
9995
9996
9997
9998
9999
```
#### 2 使用新的函数将迭代变量作为参数传入
``` golang
func main(){
    wg := sync.WaitGroup{}
    wg.Add(10000)
    for i:=0 ; i<10000 ; i++ {
        go func(j int) {
            defer  wg.Done()
            fmt.Println(j)
        }(i)
    }
    wg.Wait()
}
```
``` bash
go run share_mem.go > share_mem | sort -u > share_mem_2
```
``` text
运行结果：
9995
9996
9997
9998
9999
```



