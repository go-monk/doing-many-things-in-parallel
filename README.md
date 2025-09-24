Computers (not people!) are good at doing many things quickly. Very many things and very quickly. For example, here's a function that does something: it sleeps for a second:

```go
func doSomething() {
    time.Sleep(time.Second)
}
```

## Doing many things

When you want to do something `n` times you can do it like this:

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

As expected, around 10 seconds ... What if we wanted to do this something a million times? That would take at least a million seconds which is almost a dozen of days. Wait, what is this, computers are supposed to be fast!

## Doing many things fast

You can make them faster by programming them to do things concurrently instead of sequentially - one after another - as above. And if you have multiple CPUs (which nowadays you almost certainly have) these things are done in parallel (definitely something a man should not try to simulate - read Johann Hari's book Lost Focus to learn more)!

You can run many things in Go concurrently by spawning new **goroutines** like this:

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

Note that starting from Go 1.25, you can use the new `wg.Go()` to simplify like this:

```go
func doManyThings(n int) {
    var wg sync.WaitGroup
    for range n {
        wg.Go(func() {      // Go 1.25+: combines Add(1) + go func()
            doSomething()   // no need for defer wg.Done() - handled automatically
        })
    }
    wg.Wait()
}
```

So how long does it take now to do million times something that takes one second? On my laptop it's less than three seconds:

```sh
$ time go run main.go 

real    0m2.667s
```

That's much faster then twelve days!

## Doing many things producing output fast

When doing something we often want to see some output. If nothing else we at least want to know whether the task was done without problems:

```go
// Now doSomething returns an error value. It's nil when all is good.
func doSomething() error {
    time.Sleep(time.Second)
    if rand.Intn(100) == 0 { // circa one percent of tasks will fail
        return errors.New("we got a problem")
    }
    return nil
}
```

Now, how do we get output from tasks being concurrently executing on goroutines? For this we use **channels**:

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

As you can see we don't need a `sync.WaitGroup` counter anymore. That's because we do `n` loops (via `for range n`) one more time to receive all the values from the channel `ch`. This second loop effectively waits for all goroutines to complete - each goroutine sends exactly one value to the channel, so by reading `n` values from the channel, we know that all `n` goroutines have finished their work. The buffered channel (with capacity `n`) ensures that all goroutines can send their results without blocking, even if we haven't started reading from the channel yet.

To learn more about concurrency in Go you can go on a [tour](https://go.dev/tour/concurrency/1) or read a [book](https://www.gopl.io/).
