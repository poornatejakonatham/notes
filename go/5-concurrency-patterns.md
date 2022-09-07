# 5 Concurrency Patterns in Golang

This article will cover 5 simple concurrency patterns which are often used in Golang

## 1. `for-select` Pattern

This is a fundamental pattern. It is typically used to read data from multiple channels.

```go
var c1, c2 <-chan int

for { // Either loop infinitely or range over something
    select {
    case <-c1: // Do some work with channels
    case <-c2:
    default: // auto run if other cases are not ready
    }

    // do some work
}
```

The `select` statement looks like `switch` one, but its behavior is different. All cases are considered simultaneously & have equal chance to be selected. If none of the cases are ready to run, the entire select statement blocks.

## 2. `done` Channel Pattern

Goroutine is not garbage collected; hence, it is likely to be leaked.

```go
go func() {
// <operation that will block forever>
// => Go routine leaks
}()
// Do work
```

To avoid leaking, Goroutine should be cancelled whenever it is told to do. A parent Goroutine needs to send cancellation signal to its child via a read-only channel named `done`. By convention, it is set as the 1st parameter.

This pattern is also utilized a lot in other patterns.

```go
// Child goroutine
doWork(<-done chan interface {}, other_params) <- terminated chan interface{} {
    terminated := make(chan interface{}) // to tell outer that it has finished
    defer close(terminated)

    for {
        select: {
            case: //do your work here
            case <- done:
                return
        }
        // do work here
    }

    return terminated
}

// Parent goroutine
done := make(chan interface{})
terminated := doWork(done, other_args)

// do sth
// then tell child to stop
close (done)

// wait for child finish its work
<- terminated
```

## 3. `or-channel` Pattern

This pattern aims to combine multiple `done` channels into one `agg_done`; it means that if one of a `done` channel is signaled, the whole `agg_done` channel is also closed. Yet, we do not know number of `done` channels during runtime in advanced.

`or-channel` pattern can do so by using goroutine & recursion.

```go
// return agg_done channel
var or func(channels ... <-chan interface{}) <- chan interface{}

or = func(channels ...<-chan interface{}) <-chan interface{} {
    // base cases
    switch len(channels) {
        case 0: return nil
        case 1: return channels[0]
    }

    orDone := make(chan interface{})

    go func() {
        defer close(orDone)

        switch len(channels) {
            case 2:
                select {
                    case <- channels[0]:
                    case <- channels[1]:
                }
            default:
                select {
                    case <- channels[0]:
                    case <- channels[1]:
                    case <- channels[2]:
                    case <- or(append(channels[3:], orDone)...): // * line
                }

        }

    }
    return orDone
}
```

line * makes the upper & lower recursive function depends on each other like a tree. The upper injects its own `orDone` channel into the lower. Then the lower also return its own `orDone` to the upper.

If any `orDone` channel closes, the upper & lower both are notified.

## 4. `tee Channel Pattern

This pattern aims to split values coming from a channel into 2 others. So that we can dispatch them into two separate areas of our codebase.

```go
tee := func(
    done <- chan interface{},
    in <- chan interface{},
) (<- chan interface, <- chan interface) {
    out1 := make(chan interface{})
    out2 := make(chan interface{})

    go func() {
        defer close(out1)
        defer close(out2)

        //shadow outer variable
        var out1, out2 = out1, out2
        for val := range orDone(done, in) {
            for i := 0; i < 2; i ++ { //make sure 2 channels received same value
                select {
                case <- done:
                case out1<- val:
                    out1 = nil //stop this channel from being received
                case out2<-val:
                    out2 = nil
                }
            }
        }
    }()
    return out1, out2
}
```

## 5. `bridge` Channel Pattern

Reading values from channel of channels (`<-chan <-chan interface{}`) can be cumbersome. Hence, this pattern aims to merge all values into 1 channel, so that the consumer jobs is much easier.

```go
bridge := func(
    done <- chan interface{},
    chanStream <- <- interface{},
) <- chan interface{} {
    valStream := make(chan interface{})

    go func() {
        defer close(valStream)

        for {
            var stream <- chan interface{}
            select {
            case maybeStream, ok := <-chanStream
                if ok == false {
                    return
                }
                stream = maybeStream
            case <- done:
                return
            }

            for val := range orDone(done, stream){
                select{
                case valStream <- val:
                case <- done:
                }
            }
        }
    }()
    return valStream
}
```

**References**: Concurrency in Go: Tools and Techniques for Developers - Book by Katherine Cox-Buday