
原创文章，欢迎转载，转载请注明出处，谢谢。




---


# 0\. 前言


第八讲介绍了当 goroutine 运行时间过长会被抢占的情况。这一讲继续看 goroutine 执行系统调用时间过长的抢占。


# 1\. 系统调用时间过长的抢占


看下面的示例：



```
func longSyscall() {
	timeout := syscall.NsecToTimeval(int64(5 * time.Second))
	fds := make([]syscall.FdSet, 1)

	if _, err := syscall.Select(0, &fds[0], nil, nil, &timeout); err != nil {
		fmt.Println("Error:", err)
	}

	fmt.Println("Select returned after timeout")
}

func main() {
	threads := runtime.GOMAXPROCS(0)
	for i := 0; i < threads; i++ {
		go longSyscall()
	}

	time.Sleep(8 * time.Second)
}

```

longSyscall goroutine 执行一个 5s 的系统调用，在系统调用过程中，sysmon 会监控 longSyscall，发现执行系统调用过长，会对其抢占。


回到 `sysmon` 线程看它是怎么抢占系统调用时间过长的 goroutine 的。



```
func sysmon() {
    ...
    idle := 0 // how many cycles in succession we had not wokeup somebody
    delay := uint32(0)
    ...

    for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay)

        ...
        // retake P's blocked in syscalls
		// and preempt long running G's
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
        ...
    }
}

```

类似于运行时间过长的 goroutine，调用 `retake` 进行抢占：



```
func retake(now int64) uint32 {
	n := 0
	lock(&allpLock)
	for i := 0; i < len(allp); i++ {
		pp := allp[i]
		if pp == nil {
			continue
		}
		pd := &pp.sysmontick
		s := pp.status
		sysretake := false

        if s == _Prunning || s == _Psyscall {                           // goroutine 处于 _Prunning 或 _Psyscall 时会抢占
			// Preempt G if it's running for too long.
			t := int64(pp.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {   
                // 对于 _Prunning 或者 _Psyscall 运行时间过长的情况，都会进入 preemptone
                // preemptone 我们在运行时间过长的抢占中介绍过，它主要设置了 goroutine 的标志位
                // 对于处于系统调用的 goroutine，这么设置并不会抢占。因为线程一直处于系统调用状态           
				preemptone(pp)                                          
				// In case of syscall, preemptone() doesn't
				// work, because there is no M wired to P.
				sysretake = true
			}
		}

        if s == _Psyscall {                                             
            // Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
            // P 处于系统调用之中，需要检查是否需要抢占
            // syscalltick 用于记录系统调用的次数，在完成系统调用之后加 1
			t := int64(pp.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
                // pd.syscalltick != pp.syscalltick，说明已经不是上次观察到的系统调用了，  
                // 而是另外一次系统调用，需要重新记录 tick 和 when 值
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}

            // On the one hand we don't want to retake Ps if there is no other work to do,
			// but on the other hand we want to retake them eventually
			// because they can prevent the sysmon thread from deep sleep.
            // 如果满足下面三个条件的一个则执行抢占：
            // 1. 线程绑定的本地队列中有可运行的 goroutine
            // 2. 没有无所事事的 P（表示大家都挺忙的，那就不要执行系统调用那么长时间占资源了）
            // 3. 执行系统调用时间超过 10ms 的
			if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}

            // 下面是执行抢占的逻辑
            unlock(&allpLock)
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before the CAS.
			// Otherwise the M from which we retake can exit the syscall,
			// increment nmidle and report deadlock.
			incidlelocked(-1)
			if atomic.Cas(&pp.status, s, _Pidle) {                  // 将 P 的状态更新为 _Pidle
				n++                                                 // 抢占次数 + 1
				pp.syscalltick++                                    // 系统调用抢占次数 + 1              
				handoffp(pp)                                        // handoffp 抢占
			}
			incidlelocked(1)
			lock(&allpLock)
        }
    }
    unlock(&allpLock)
	return uint32(n)
}

```

进入 `handoffp`：



```
// Hands off P from syscall or locked M.
// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func handoffp(pp *p) {
    // if it has local work, start it straight away
    // 这里如果 P 的本地有工作（goroutine），或者全局有工作的话
    // 将 P 和其它线程绑定，其它线程指的是不是执行系统调用的那个线程
    // 执行系统调用的线程不需要 P 了，这时候把 P 释放出来，算是资源的合理利用，相比于线程，P 是有限的
	if !runqempty(pp) || sched.runqsize != 0 {
		startm(pp, false, false)
		return
	}

    ...
    // no local work, check that there are no spinning/idle M's,
	// otherwise our help is not required
	if sched.nmspinning.Load()+sched.npidle.Load() == 0 && sched.nmspinning.CompareAndSwap(0, 1) { // TODO: fast atomic
		sched.needspinning.Store(0)
		startm(pp, true, false)
		return
	}

    ...
    // 判断全局队列有没有工作要处理
    if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}

    ...
    // 如果都没有工作，那就把 P 放到全局空闲队列中
    pidleput(pp, 0)
	unlock(&sched.lock)
}

```

可以看到抢占系统调用过长的 goroutine，这里抢占的意思是释放系统调用线程所绑定的 P，抢占的意思不是不让线程做系统调用，而是把 P 释放出来。(由于前面设置了这个 goroutine 的 stackguard0，类似于 [运行时间过长 goroutine 的抢占](https://github.com) 的流程还是会走一遍的)。


我们看一个示意图可以更直观清晰的了解这个过程：


![image](https://img2024.cnblogs.com/blog/1498760/202409/1498760-20240916120055265-574615154.png)


`handoff` 结束之后，增加抢占次数 n，`retake` 返回：



```
func sysmon() {
	...
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20                  // 如果 idle == 0，表示 sysmon 需要打起精神来，要隔 20us 监控一次
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2                  // 如果 idle 大于 50，表示循环了 50 次都没有抢占，sysmon 将加倍休眠，比较空，sysmon 也不浪费资源，先睡一会
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000           // 当然，不能无限制睡下去。最大休眠时间设置成 10ms
		}

        if retake(now) != 0 {               
			idle = 0                    // 有抢占，则 idle = 0，表示 sysmon 要忙起来
		} else {
			idle++                      // 没有抢占，idle + 1
		}
    ...
    }
    ...
}

```

# 2\. 小结


本讲介绍了系统调用时间过长引起的抢占。下一讲将继续介绍异步抢占。




---


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
