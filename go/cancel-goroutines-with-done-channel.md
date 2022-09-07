# Cancelling all running goroutines with a done channel as soon as one sends to channel in Golang

There are many options to cancel goroutines but this example shows us how we can use a channel for such purpose. We are going to run three goroutines and as soon as one of them writes to the "done" channel, all the others will be terminated. Each goroutine is meant to print three times.

```go
package main

import (
	"log"
	"time"
)

func main() {
	done := make(chan struct{})

	go run(done, 1)
	go run(done, 2)
	time.Sleep(time.Second) // Cause the last one to not complete all three jobs
	go run(done, 3)

	<-done
}

func run(done chan<- struct{}, id int) {
	i := 1

	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// Do something here
			log.Println(id, "- done", i)

			i++
			if i == 4 {
				done<- struct{}{}
			}
		}
	}
}
```

As seen below third goroutine didn't have a change run three times.

```bash
2020/06/16 22:32:24 2 - done 1
2020/06/16 22:32:24 1 - done 1
2020/06/16 22:32:25 2 - done 2
2020/06/16 22:32:25 1 - done 2
2020/06/16 22:32:25 3 - done 1
2020/06/16 22:32:26 1 - done 3
2020/06/16 22:32:26 2 - done 3
```
