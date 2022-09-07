# Don't communicate by sharing memory; share memory by communicating

This statement applies to concurrent programming. It states that **message passing** should be the preferred approach to communicate not just between processes, but also between threads. The reason is that it is very difficult to properly implement systems who communicate by writing to shared memory. The most common problem is race condition which occur when an actor (could be any one of threads, processes, machines) modifies shared state when other actors don't expect it.

Message passing avoids that problem entirely: ideally, there is no shared state. The only thing actors can do is to communicate. This way their actions can be coordinated. This could be, among other things, access to shared resources, such as a block of memory.

## Introduction

This post will demonstrate how to do inter-thread communication in Golang concurrent programming. Generally speaking, there are two ways for this fundamental question: *shared memory* and *message passing*. You'll see how to do these in Golang based on a case study and some tricky problems around it.

## Background

Golang is a new popular and powerful programming language that aims to provide a simple, efficient, and safe way to build multi-threaded software.

Concurrent programming is one of Go's main selling points. Go uses a concept called **goroutine** as its concurrency unit. Goroutine is a complex but interesting topic, you can find many articles about it online, This post will not cover concepts about it in detail. Simply speaking, goroutine is a user-space level thread which is lightweight and easy to create.

As mentioned above, one of the complicated problems when we do concurrent programming is that inter-thread (or inter-goroutine) communication is very error-prone. In Golang, it provides frameworks for both *shared memory* and *message passing*. However, it encourages the use of channels over shared memory.

You'll see how both of these methods in Golang based on the following case.

## Case study

The example is very simple: sums a collection (10 million) of integers. In fact this example is based on this good [article](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html). It used the *shared memory* way to realize the communication between goroutines. I expand this example and implemented the *message passing* way to show the difference.

## Shared Memory

Go supports traditional shared memory accesses among goroutines. You can use various traditional synchronization primitives such as *lock/unlock* (Mutex), *condition variable* (Cond) and *atomic* read/write operations(atomic).

In the following implementation, you can see Go uses `WaitGroup` to allow multiple goroutines to do their tasks before a waiting goroutine. This usage is very similar to `pthread_join` in C.

Goroutines are added to a WaitGroup by calling `Add` method. And the goroutines in a WaitGroup call `Done` method to notify their completion, while a goroutine make a call to `Wait` method to wait for all goroutines' completion.

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"sync"
	"sync/atomic"
)

func main() {
	numbers := generateList(1e7)

	fmt.Println("add: ", add(numbers))
	fmt.Println("add concurrent: ", addConcurrent(runtime.NumCPU(), numbers))
}

func generateList(totalNumbers int) []int {
	numbers := make([]int, totalNumbers)
	for i := 0; i < totalNumbers; i++ {
		numbers[i] = rand.Intn(totalNumbers)
	}
	return numbers
}

func add(numbers []int) int {
	var v int
	for _, n := range numbers {
		v += n
	}
	return v
}

func addConcurrent(goroutines int, numbers []int) int {
	var v int64
	totalNumbers := len(numbers)
	lastGoroutine := goroutines - 1

	strid := totalNumbers / goroutines

	var wg sync.WaitGroup
	wg.Add(goroutines)

	for g := 0; g < goroutines; g++ {
		go func(g int) {
			start := g * strid
			end := start + strid
			if g == lastGoroutine {
				end = totalNumbers
			}

			var lv int
			for _, n := range numbers[start:end] {
				lv += n
			}
			atomic.AddInt64(&v, int64(lv))
			wg.Done()
		}(g)
	}

	wg.Wait()
	return int(v)
}
```

In the example above the `int64` variable `v` is shared across goroutines. When this variable needs to be updated, an atomic operation was done by calling `atomic.AddInt64()` method to avoid race condition and nondeterministic result.

That's how shared memory across goroutines works in Golang. Let's go to message passing way in next section.

## Message Passing

In Golang world, there is one sentence is famous:

**Don't communicate by sharing memory; share memory by communicating**

For that, `Channel`(chan) is introduced in Go as a new concurrency primitive to send data across goroutines. This is also the way Golang recommended you to follow.

So the concurrent program to sum 10 million integers based on `Channel` goes as below:

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
)

func main() {
	var result int64
	numbers := generateList(1e7)

	goroutines := runtime.NumCPU()
	lastGoroutine := goroutines - 1

	totalNumbers := len(numbers)
	strid := totalNumbers / goroutines

	c := make(chan int)

	for g := 0; g < goroutines; g++ {
		start := g * strid
		end := start + strid
		if g == lastGoroutine {
			end = totalNumbers
		}

		go add(numbers[start:end], c)
	}

	for l := range c {
		result += int64(l)
	}

	fmt.Println("add concurrent: ", result)
}

func generateList(totalNumbers int) []int {
	numbers := make([]int, totalNumbers)
	for i := 0; i < totalNumbers; i++ {
		numbers[i] = rand.Intn(totalNumbers)
	}
	return numbers
}

func add(numbers []int, c chan int) {
	var v int
	for _, n := range numbers {
		v += n
	}
	c <- v
}
```

To create a typed channel, you can call `make` method. In this case, since the value we need to pass is an integer, so we create an int type channel with `c := make(chan int)`. To read and write data to that channel, you can use `<-` operator. For example, in the `add` goroutine, when we get the sum of integers, we use `c <- v` to send data to the channel.

To read data from the channel in the main goroutine, we use a build-in method `range` in Golang which can iterate through data structure like slice, map and channel.

That's it. Simple and beautiful.

### Hit the Deadlock

Let's build and run the above solution. You'll get an error message as following:

```bash
$ fatal error: all goroutines are asleep - deadlock!
```

The *deadlock* issue occurs because of these two reasons. Firstly by default sends and receives to a channel are blocking. When a data is send to a channel, the control in that goroutine is blocked at the send statement until some other Goroutine reads from the channel. Similarly when data is read from a channel, the read is blocked until some Goroutine writes data to that channel. Secondly, `range` only stops when the channel is closed. In this case, each `add` Goroutine send only one value to the channel but didn’t close the channel. And the main Goroutine keeps waiting for something to be written (in fact, it can read 4 values, but after that it doesn’t stop and keep waiting for more data). So all of the Goroutines are blocked and none of them can continue the execution. Then hit a deadlock.

### Fix the Deadlock

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
)

func main() {
	var result int64
	numbers := generateList(1e7)

	goroutines := runtime.NumCPU()
	lastGoroutine := goroutines - 1

	totalNumbers := len(numbers)
	strid := totalNumbers / goroutines

	c := make(chan int)

	for g := 0; g < goroutines; g++ {
		start := g * strid
		end := start + strid
		if g == lastGoroutine {
			end = totalNumbers
		}

		go add(numbers[start:end], c)
	}

	for j := 0; j < goroutines; j++ {
		result += int64(<-c)
	}

	fmt.Println("add concurrent: ", result)
}

func generateList(totalNumbers int) []int {
	numbers := make([]int, totalNumbers)
	for i := 0; i < totalNumbers; i++ {
		numbers[i] = rand.Intn(totalNumbers)
	}
	return numbers
}

func add(numbers []int, c chan int) {
	var v int
	for _, n := range numbers {
		v += n
	}
	c <- v
}
```

Let use the manual `for` loop, in each iteration we read the value from the channel and sum together. Run it again. The deadlock is resolved.