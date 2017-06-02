---
layout: post_layout
title: Go：使用channel共享内存
time: 2017年6月3日 星期六
location: 北京
pulished: true
---

Go的口号：不要通过共享内存来通信，而应通过通信来共享内存

<!--break-->

Talk is cheap, I will show you the code.

```
func main() {
	c1 := make(chan int)
	quit := make(chan int)
	test := 0
	a := [...]int{1, 2, 3}
	for _, v := range a {
		go func(v int) {
			for {
				select {
				case my := <-c1:
					my++
					fmt.Printf("id: %d now count is %d \n", v, my)
					c1 <- my
				case <-quit:
					fmt.Printf("id: %d return \n", v)
					return
				}
			}
		}(v)
	}
	c1 <- test
	time.Sleep(5000)
	fmt.Printf("we get the count: %d \n", test)
	for _, v := range a {
		quit <- v
	}
	fmt.Printf("all is over \n")
	return
	//panic("hello")
}

```

如代码所示，3个Go程对test进行计数。使用c1传递消息，使用quit传递退出消息。

这段代码有一个问题。
在调试过程中，出现了以下的错误信息。


```
id: 1 now count is 495 
id: 2 now count is 496 
id: 3 now count is 497 
id: 1 return 
id: 2 return 
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/home/du/go-learning/test.go:33 +0x1e4

goroutine 7 [chan send]:
main.main.func1(0xc42006a060, 0xc42006a0c0, 0x3)
	/home/du/go-learning/test.go:22 +0x1c7
created by main.main
	/home/du/go-learning/test.go:27 +0x11a
exit status 2
```
可以看出，goroutine 1，7 分别卡在了发送上。即
```
13行： c1 <- my
25行： quit <- v
```

非缓冲的chan必须接收方和发送方都准备好，才能实现通信。
当ID为1，2的Go程退出后，c1没有接收方了，因此死锁。
最后一个quit也没有接收方，因此死锁。

刚开始，主Go程给c1中发送数据test，使整个流程流动起来，最终也要拿出test,终止流程。

```
    c1 <- test  
    time.Sleep(5000)
    test  = <-c1   //增加的代码

最终输出：
	id: 3 now count is 1 
	id: 1 now count is 2 
	id: 3 now count is 3 
	id: 1 now count is 4 
	id: 2 now count is 5 
	we get the count: 5 
	all is over
```

