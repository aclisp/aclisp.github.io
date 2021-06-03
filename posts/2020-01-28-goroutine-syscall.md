# Go 的调度器如何处理系统调用

> So the Go runtime and scheduler actually have two ways of handling blocking system calls, a pessimistic way for system calls that are expected to be slow and an optimistic way for ones that are expected to be fast. The pessimistic system call path implements the straightforward approach where the runtime actively releases the P before the system call, attempts to get it back afterward, and parks itself if it can't. The optimistic system call path doesn't release the P; instead, it sets a special P state flag and just makes the system call. A special internal goroutine, the sysmon goroutine, then comes along periodically and looks for P's that have been sitting in this 'making a system call' state for too long and steals them away from the goroutine making the system call. When the system call returns, the runtime code checks to see if its P has been stolen out from underneath it, and if it hasn't it can just go on (if the P has been stolen, the runtime tries to get another P and then may have to park your goroutine).

## goID 如何生成

```
// Create a new g running fn with narg bytes of arguments starting
// at argp. callerpc is the address of the go statement that created
// this. The new g is put on the queue of g's waiting to run.
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
    ...
    if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
    ...
}
```

## 什么是 spinning threads

一个例子是 `findrunnable` 不停的找可运行的 G

```
    for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
				return gp, false
			}
		}
	}
```

这个过程满足一定条件才触发，参考：[Spinning threads](https://rakyll.org/scheduler/)

## 参考文档

1. [The Go runtime scheduler's clever way of dealing with system calls](https://utcc.utoronto.ca/~cks/space/blog/programming/GoSchedulerAndSyscalls)
1. [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
1. [Go 1.5 源码剖析.pdf](https://github.com/qyuhen/book)
1. [goroutine的生老病死](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)
