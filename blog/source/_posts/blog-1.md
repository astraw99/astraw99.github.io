---
title: Go嵌套并发实现EDM，附坑点分析#1
---
看着身边优秀的小伙伴们早就开始写博客，自己深感落后，还好迟做总比不做好，勉励自己见贤思齐。趁着年前最后一个周末，阳光正好，写下第一篇博客，为2019年开个头，以期完成今年为自己立下的flags。



从PHPer转Gopher，很大一个原因就是业务对性能和并发的持续需求，另一个主要原因就是Go语言原生的并发特性，可以在提供同等高可用的能力下，使用更少的机器资源，节约可观的成本。因此本文就结合自己在学习Go并发的实战demo中，把遇到的一些坑点写下来，共享进步。



## 1. 在Go语言中实现并发控制，目前主要有三种方式：

a) `Channel` - 分为无缓冲、有缓冲通道;

b) `WaitGroup` - sync包提供的goroutine间的同步机制;

c) `Context` - 在调用链不同goroutine间传递和共享数据;

本文demo中主要用到了前两种，基本使用请查看官方文档。



## 2. Demo需求与分析：

需求：实现一个EDM的高效邮件发送：需要支持多个国家(可以看成是多个任务)，需要记录每条任务发送的状态(当前成功、失败条数)，需要支持可暂停(stop)、重新发送(run)操作。



分析：从需求可以看出，在邮件发送中可以通过并发实现多个国家(多个任务)并发、单个任务分批次并发实现快速、高效EDM需求。



## 3. Demo实战源码：

3.1 main.go
```go
package main

import (
  "bufio"
  "fmt"
  "io"
  "log"
  "os"
  "strconv"
  "sync"
  "time"
)

var (
  batchLength = 20
  wg          sync.WaitGroup
  finish      = make(chan bool)
)

func main() {
  startTime := time.Now().UnixNano()

  for i := 1; i <= 3; i++ {
    filename := "./task/edm" + strconv.Itoa(i) + ".txt"
    start := 60

    go RunTask(filename, start, batchLength)
  }

  // main 阻塞等待goroutine执行完成
  fmt.Println(<-finish)

  fmt.Println("finished all tasks.")

  endTime := time.Now().UnixNano()
  fmt.Println("Total cost(ms):", (endTime-startTime)/1e6)
}

// 单任务
func RunTask(filename string, start, length int) (retErr error) {
  for {
    readLine, err := ReadLines(filename, start, length)
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      fmt.Println(err)
      retErr = err
      break
    }

    fmt.Println("current line:", readLine)

    start += length

    // 等待一批完成才进入下一批
    //wg.Wait()
  }

  wg.Wait()
  finish <- true

  return retErr
}
```

注意上面`wg.Wait()`的位置(下面有讨论)，在`finish channel`之前，目的是为了等待子`goroutine`运行完，再通过一个无缓冲通道`finish`通知`main goroutine`，然后`main`运行结束。



func ReadLines()读取指定行数据：
```go
// 读取指定行数据
func ReadLines(filename string, start, length int) (line int, retErr error) {
  fmt.Println("current file:", filename)

  fileObj, err := os.Open(filename)
  if err != nil {
    panic(err)
  }
  defer fileObj.Close()

  // 跳过开始行之前的行-ReadString方式
  startLine := 1
  endLine := start + length
  reader := bufio.NewReader(fileObj)
  for {
    line, err := reader.ReadString(byte('\n'))
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      log.Fatal(err)
      retErr = err
      break
    }

    if startLine > start && startLine <= endLine {
      wg.Add(1)
      // go并发执行
      go SendEmail(line)
      if startLine == endLine {
        break
      }
    }

    startLine++
  }

  return startLine, retErr
}

// 模拟邮件发送
func SendEmail(email string) error {
  defer wg.Done()

  time.Sleep(time.Second * 1)
  fmt.Println(email)

  return nil
}
```

运行上面`main.go`，3个任务在1s内并发完成所有邮件(`./task/edm1.txt`中一行表示一个邮箱)发送。
```
true

finished all tasks.

Total cost(ms): 1001
```


那么问题来了：没有实现分批每次并发`batchLength = 20`，因为如果不分批发送，只要其中某个任务或某一封邮件出错了，那下次重新run的时候，会不知道哪些用户已经发送过了，出现重复发送。而分批发送即使中途出错了，下一次重新run可从上次出错的end行开始，最多是`[start - end]`一个`batchLength` 发送失败，可以接受。



于是，将倒数第5行`wg.Wait()`注释掉，倒数第8行注释打开，如下：
```go
// 单任务
func RunTask(filename string, start, length int) (retErr error) {
  for {
    readLine, err := ReadLines(filename, start, length)
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      fmt.Println(err)
      retErr = err
      break
    }

    fmt.Println("current line:", readLine)

    start += length

    // 等待一批完成才进入下一批
    wg.Wait()
  }

  //wg.Wait()
  finish <- true

  return retErr
}
```

运行就报错：
```
panic: sync: WaitGroup is reused before previous Wait has returned
```


提示`WaitGroup`在`goroutine`之间重用了，虽然是全局变量，看起来是使用不当。怎么调整呢？


3.2 main.go
```go
package main

import (
  "bufio"
  "fmt"
  "io"
  "log"
  "os"
  "strconv"
  "sync"
  "time"
)

var (
  batchLength = 10
  outerWg     sync.WaitGroup
)

func main() {
  startTime := time.Now().UnixNano()

  for i := 1; i <= 3; i++ {
    filename := "./task/edm" + strconv.Itoa(i) + ".txt"
    start := 60

    outerWg.Add(1)
    go RunTask(filename, start, batchLength)
  }

  // main 阻塞等待goroutine执行完成
  outerWg.Wait()

  fmt.Println("finished all tasks.")

  endTime := time.Now().UnixNano()
  fmt.Println("Total cost(ms):", (endTime-startTime)/1e6)
}

// 单任务
func RunTask(filename string, start, length int) (retErr error) {
  for {
    isFinish := make(chan bool)
    readLine, err := ReadLines(filename, start, length, isFinish)
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      fmt.Println(err)
      retErr = err
      break
    }

    // 等待一批完成才进入下一批
    fmt.Println("current line:", readLine)
    start += length
    <-isFinish

    // 关闭channel，释放资源
    close(isFinish)
  }

  outerWg.Done()

  return retErr
}
```

从上面可以看出：调整的思路是外层用`WaitGroup`控制，里层用`channel` 控制，执行又报错 : (


```
fatal error: all goroutines are asleep - deadlock!



goroutine 1 [semacquire]:

sync.runtime_Semacquire(0x55fe7c)

	/usr/local/go/src/runtime/sema.go:56 +0x39

sync.(*WaitGroup).Wait(0x55fe70)

	/usr/local/go/src/sync/waitgroup.go:131 +0x72

main.main()

	/home/work/data/www/docker_env/www/go/src/WWW/edm/main.go:31 +0x1ab



goroutine 5 [chan send]:

main.ReadLines(0xc42001c0c0, 0xf, 0x3c, 0xa, 0xc42008e000, 0x0, 0x0, 0x0)
```


仔细检查，发现上面代码中定义的`isFinish` 是一个无缓冲`channel`，在发邮件`SendMail()` 子协程没有完成时，读取一个无数据的无缓冲通道将阻塞当前`goroutine`，其他`goroutine`也是一样的都被阻塞，这样就出现了`all goroutines are asleep - deadlock!`



于是将上面代码改为有缓冲继续尝试：
```go
isFinish := make(chan bool, 1)
// 读取指定行数据
func ReadLines(filename string, start, length int, isFinish chan bool) (line int, retErr error) {
  fmt.Println("current file:", filename)

  // 控制每一批发完再下一批
  var wg sync.WaitGroup

  fileObj, err := os.Open(filename)
  if err != nil {
    panic(err)
  }
  defer fileObj.Close()

  // 跳过开始行之前的行-ReadString方式
  startLine := 1
  endLine := start + length
  reader := bufio.NewReader(fileObj)
  for {
    line, err := reader.ReadString(byte('\n'))
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      log.Fatal(err)
      retErr = err
      break
    }

    if startLine > start && startLine <= endLine {

      wg.Add(1)
      // go并发执行
      go SendEmail(line, wg)
      if startLine == endLine {
        isFinish <- true
        break
      }
    }

    startLine++
  }

  wg.Wait()

  return startLine, retErr
}

// 模拟邮件发送
func SendEmail(email string, wg sync.WaitGroup) error {
  defer wg.Done()

  time.Sleep(time.Second * 1)
  fmt.Println(email)

  return nil
}
```

运行，又报错了 : (


```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:

sync.runtime_Semacquire(0x55fe7c)

	/usr/local/go/src/runtime/sema.go:56 +0x39

sync.(*WaitGroup).Wait(0x55fe70)
```


这次提示有点不一样，看起来是里层的`WaitGroup` 导致了死锁，继续检查发现里层`wg` 是值传递，应该使用指针传引用。


```go
// go并发执行
go SendEmail(line, wg)
```

最后修改代码如下：
```go
// 读取指定行数据
func ReadLines(filename string, start, length int, isFinish chan bool) (line int, retErr error) {
  fmt.Println("current file:", filename)

  // 控制每一批发完再下一批
  var wg sync.WaitGroup

  fileObj, err := os.Open(filename)
  if err != nil {
    panic(err)
  }
  defer fileObj.Close()

  // 跳过开始行之前的行-ReadString方式
  startLine := 1
  endLine := start + length
  reader := bufio.NewReader(fileObj)
  for {
    line, err := reader.ReadString(byte('\n'))
    if err == io.EOF {
      fmt.Println("Read EOF:", filename)
      retErr = err
      break
    }
    if err != nil {
      log.Fatal(err)
      retErr = err
      break
    }

    if startLine > start && startLine <= endLine {

      wg.Add(1)
      // go并发执行
      go SendEmail(line, &wg)
      if startLine == endLine {
        isFinish <- true
        break
      }
    }

    startLine++
  }

  wg.Wait()

  return startLine, retErr
}

// 模拟邮件发送
func SendEmail(email string, wg *sync.WaitGroup) error {
  defer wg.Done()

  time.Sleep(time.Second * 1)
  fmt.Println(email)

  return nil
}
```

赶紧运行一下，这次终于成功啦 : )


```
current line: 100

current file: ./task/edm2.txt

Read EOF: ./task/edm2.txt

Read EOF: ./task/edm2.txt

finished all tasks.

Total cost(ms): 4003
```


每个任务模拟的是100行，从第60行开始运行，四个任务并发执行，每个任务分批内再次并发，并且控制了每一批次完成后再进行下一批，所以总运行时间约4s，符合期望值。完整源码请阅读原文或移步GitHub：[https://github.com/astraw99/edm](https://github.com/astraw99/edm)



## 4. 小结：

本文通过两层嵌套Go 并发，模拟实现了高性能并发EDM，具体的一些出错行控制、任务中断与再次执行将在下次继续讨论，主要逻辑已跑通，几个坑点小结如下：



**a) WaitGroup 一般用于main 主协程等待全部子协程退出后，再优雅退出主协程；嵌套使用时注意wg.Wait()放的位置；**

**b) 合理使用channel，无缓冲chan将阻塞当前goroutine，有缓冲chan在cap未满的情况下不会阻塞当前goroutine，使用完记得释放chan资源；**

**c) 注意函数间传值或传引用(本质上还是传值，传的指针的指针内存值)的合理使用；**




后记：第一篇博客写到这里差不多算完成了，一不小心一个下午就过去了，写的逻辑、可读性可能不太好请见谅，欢迎留言批评指正。感谢您的阅读。




![稻草人生](http://upload-images.jianshu.io/upload_images/6925317-ba12da4661333423?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

