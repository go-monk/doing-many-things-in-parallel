Computers (not people!) are very good at doing many things quickly. Very many things and very quickly. For example, here's a function that does something; it sleeps for a second:

```go
func doSomething() {
    time.Sleep(time.Second)
}
```

When you want to do something many times you can do it like this:

```go
func doManyThings(n int) {
    for range n {
        doSomething()
    }
}

func main() {
    doManyThings(10)
}
```

But how long does it take to do this something, let's say ten times?

```sh
$ time go run main.go 

real    0m10.256s
```

As expected, around 10 seconds ... What if we wanted to do this something a million times? That would take million seconds which is almost a dozen of days. What is this, computers are supposed to be fast!

You can make them faster by programming them to do things concurrently, i.e. not sequentially (one after another) as above. And if you have multiple CPUs (which nowadays you almost certainly have) these things are done in parallel (definitely something a man should not try to simulate)!

You can run many things in Go concurrently like this:

```go
func doManyThings(n int) {
    var wg sync.WaitGroup   // create concurrency-safe counter
    for range n {
        wg.Add(1)           // increment counter by one
        go func() {         // launch function on new goroutine and continue
            defer wg.Done() // decrement counter on return from function
            doSomething()
        }()
    }
    wg.Wait()               // wait till counter is 0 (all functions are done)
}

func main() {
    doManyThings(1e6)
}
```

So how long does it take now to do million times something that takes one second? On my laptop it's less than three seconds:

```sh
$ time go run main.go 

real    0m2.667s
```

That's much faster then twelve days!

When doing something we often want to see some output. If nothing else we want to at least know whether the task was done without problems:

```go
func doSomething() error {
    time.Sleep(time.Second)
    if rand.Intn(100) == 0 { // circa one percent of tasks will fail
        return errors.New("we got a problem")
    }
    return nil
}
```

Now, how do we get output from tasks being concurrently executing on goroutines? For this we use channels:

```go
func doManyThings(n int) []error {
    ch := make(chan error, n)   // create channel of errors that can hold n items

    for range n {
        go func() {
            ch <- doSomething() // send function's output to channel
        }()
    }

    var errs []error
    for range n {
        err := <-ch             // receive value from channel and assign it to err
        if err != nil {
            errs = append(errs, err)
        }
    }
    return errs
}

func main() {
    n := 1_000_000
    errs := doManyThings(n)
    fmt.Printf("Doing %d things there where %d errors.\n", n, len(errs))
}
```

As you can see we don't need a `sync.WaitGroup` counter anymore. That's because we do `n` loops (`for range n`) one more time to receive all the values from the channel `ch`. Unbuffered channels like the one we used are useful not only for sending and receiving values but also for synchronization. Sends and receives block until the other side is ready.

To learn more about concurrency in Go you can go on a [tour](https://go.dev/tour/concurrency/1) or read a [book](https://www.gopl.io/).
