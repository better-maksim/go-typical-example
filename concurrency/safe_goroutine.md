# 安全的 Goroutine 


在实际的工作中，很多开发者会将一些异步逻辑使用 `go` 关键字创建一个 goroutine 去执行，一旦 goroutine 中发生panic，会导致整个系统退出，这是一个很严重的问题。在这篇文章中，我将在 go func(){}上编写一个带有panic恢复功能的简单包装库(包)。之后，我们可以在任何地方使用这个包来安全地执行 Goroutine。

我们将包命名为async:

代码：让我们将包命名为async:

```Go
package async

import (
    "fmt"
)

/*
 * `Execute`提供异步安全地执行函数的方法，在可能发生panic时进行恢复。
 */
func Execute(fn func()) {
    go func() {
        defer recoverPanic()
        fn()
    }()
}
/*
 *当goroutine发送panic，将错误信息打印到控制台
 */
func recoverPanic() {
    if r := recover(); r != nil {
        err, ok := r.(error)
        if !ok {
            err = fmt.Errorf("%v", r)
        }
        fmt.Println(err)
    }
}
```

现在，您可以在项目中的任何地方轻松导入这个包，并安全地调用goroutine。例如，让我们看看下面的代码

```go
async.Execute(func() {
    someUint64, err = strconv.ParseUint("7770009", 10, 16)
    if err != nil {
       panic(err)
    }
})
```
这是一个非常实用的编程技巧，能防止go协程中一些可能存在的panic未捕获导致程序奔溃。

在 go-zero 中对并发编程进行了简单的封装，其中就包括安全的 Goroutine。

```go

//代码地址: https://github.com/zeromicro/go-zero/tree/master/core/threading

package threading

import (
	"bytes"
	"runtime"
	"strconv"

	"github.com/zeromicro/go-zero/core/rescue"
)

// GoSafe runs the given fn using another goroutine, recovers if fn panics.
func GoSafe(fn func()) {
	go RunSafe(fn)
}

// RoutineId is only for debug, never use it in production.
func RoutineId() uint64 {
	b := make([]byte, 64)
	b = b[:runtime.Stack(b, false)]
	b = bytes.TrimPrefix(b, []byte("goroutine "))
	b = b[:bytes.IndexByte(b, ' ')]
	// if error, just return 0
	n, _ := strconv.ParseUint(string(b), 10, 64)

	return n
}

// RunSafe runs the given fn, recovers if fn panics.
func RunSafe(fn func()) {

	defer rescue.Recover()

	fn()
}
```

在使用 go-zero 进行开发的时候，我们可以直接调用 GoSafe 函数安全的使用 Goroutine，其原理与我们上面讲到得方法原理是一样的，不过它是 defer 关键字在函数执行完成后进行 Recover。接下来我们来看一下 Recover() 的实现.

```go

//代码地址：https://github.com/zeromicro/go-zero/blob/master/core/rescue/recover.go

package rescue

import "github.com/zeromicro/go-zero/core/logx"

// Recover is used with defer to do cleanup on panics.
// Use it like:
//  defer Recover(func() {})
func Recover(cleanups ...func()) {
	for _, cleanup := range cleanups {
		cleanup()
	}

	if p := recover(); p != nil {
		logx.ErrorStack(p)
	}
}
```

在 Recvoer 函数中首先是 `cleanups`  可以在 recover 之前执行我们想要执行的方法。

当 cleanup 执行完成后，会将 recover 的错误日志传递给日志组件打印堆栈信息。

如果你目前所处的公司使用的框架是自研的，你可以看看有没有 safe_go 的封装，如果没有建议你讲这个方法加入到你们的项目中。

## ref

1. https://www.jianshu.com/p/b7c54e2b2216
2. go-zero