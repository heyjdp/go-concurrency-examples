# golang concurrency examples

Two infinite loops, will never get to the second loop:

```go
func main() {
	count("sheep")
	count("fish")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

Run it:

```bash
go run main.go 
1 sheep
2 sheep
3 sheep
4 sheep
5 sheep
^Csignal: interrupt
```

Add a `go routine`:

```go
func main() {
	go count("sheep")
	count("fish")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

Run it:

```bash
go run main.go 
1 fish
1 sheep
2 sheep
2 fish
3 fish
3 sheep
4 sheep
4 fish
^Csignal: interrupt
```

Now they run side-by-side, equivalent of very lightweight threads

Two `go-routine`:

```go
func main() {
	go count("sheep")
	go count("fish")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

This will exit immediately, because there is no work to be done in the main function, so it will exit. The go routines need time to execute.

We can use a `wait-group` to block the thread on the go routine completing

```go
func main() {
	var wg sync.WaitGroup
	wg.Add(1) // Just a counter

	go func() {
		count("sheep")
		wg.Done() // Decrement
	}()
	wg.Wait() // main thread waits
}

func count(thing string) {
	// Don't pass wg to the function, it is not the count functions
	// responsibility to deal with the main threads concurrency
	for i := 1; i <= 5; i++ { // Count to 5
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

Using `channels` (IPC for go routines):

```go
func main() {
	c := make(chan string)
	go count("sheep", c)

	msg := <-c // This will block until there is something to be read
	fmt.Println(msg)
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ { // Count to 5
		c <- thing // Send a message to the channel
		time.Sleep(time.Millisecond * 500)
	}
}
```

Run it:

```bash
go run main.go 
sheep
```

The blocking nature of channels allows us to sync the go-routines. But we only received one message, the go-routine added 5. How about this:

```go
func main() {
	c := make(chan string)
	go count("sheep", c)

	for {
		msg := <-c // Read from the channel
		fmt.Println(msg)
	}
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ { // Count to 5
		c <- thing // Send a message to the channel
		time.Sleep(time.Millisecond * 500)
	}
}
```

Now we see this deadlock:

```bash
go run main.go 
sheep
sheep
sheep
sheep
sheep
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /home/jas/go/src/github.com/heyjdp/go-concurrency-examples/main.go:13 +0x7f
exit status 2
```

This is because the reader wants to consume input, but the go-routine has stopped producting after 5.

We can use the open/closed state of the channel to signal that we are done:

```go
func main() {
	c := make(chan string)
	go count("sheep", c)

	for {
		msg, open := <-c // Read from the channel
		if !open {
			break // If the channel is closed, break
		}
		fmt.Println(msg)
	}
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ { // Count to 5
		c <- thing // Send a message to the channel
		time.Sleep(time.Millisecond * 500)
	}
	close(c) // FLag we are done by closing the channel
	// Only senders should ever close channels
}
```

And now it works as expected:

```bash
go run main.go 
sheep
sheep
sheep
sheep
sheep
```

There is syntactic sugar for this checking whether the channel is open as follows:

```go
func main() {
	c := make(chan string)
	go count("sheep", c)

	for msg := range c {
		fmt.Println(msg)
	}
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ { // Count to 5
		c <- thing // Send a message to the channel
		time.Sleep(time.Millisecond * 500)
	}
	close(c) // Flag we are done by closing the channel
	// Only senders should ever close channels
}
```

Blocking on channels: 

```go
func main() {
	c := make(chan string)
	c <- "hello" // Blocked until someone is ready to receive
	msg := <-c // Never gets to here
	fmt.Println(msg)
}
```

Will deadlock:

```bash
go run main.go 
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /home/jas/go/src/github.com/heyjdp/go-concurrency-examples/main.go:7 +0x37
exit status 2
```

We can make a buffered channel, which will only block when the channel is full:

```go
func main() {
	c := make(chan string, 2) // Buffered channel
	c <- "hello" 
	msg := <-c 
	fmt.Println(msg)
}
```

And runs:

```bash
go run main.go 
hello
```

And we can put two things in:

```go
func main() {
	c := make(chan string, 2) // Buffered channel
	c <- "hello"
	c <- "world" // Put two things in
	msg := <-c
	fmt.Println(msg)
	msg = <-c
	fmt.Println(msg)
}
```

And it works as expected

Golang has a `select` statement. Consider the following:

```go
func main() {
	c1 := make(chan string)
	c2 := make(chan string)

	go func() {
		for {
			c1 <- "Every 500ms"
			time.Sleep(time.Millisecond * 500)
		}
	}()

	go func() {
		for {
			c2 <- "Every 2000ms"
			time.Sleep(time.Millisecond * 2000)
		}
	}()

	for {
		fmt.Println(<-c1) // Ready but...
		fmt.Println(<-c2) // ...is blocking
	}
}
```

Runs like this:

```bash
go run main.go 
Every 500ms
Every 2000ms
Every 500ms
Every 2000ms
Every 500ms
Every 2000ms
Every 500ms
Every 2000ms
Every 500ms
^Csignal: interrupt
```

So we are always blocked on the slower go-routine. Let's use select to take input from whichevevr channel is ready:

```go
func main() {
	c1 := make(chan string)
	c2 := make(chan string)

	go func() {
		for {
			c1 <- "Every 500ms"
			time.Sleep(time.Millisecond * 500)
		}
	}()

	go func() {
		for {
			c2 <- "Every 2000ms"
			time.Sleep(time.Millisecond * 2000)
		}
	}()

	for {
		select {
		case msg1 := <-c1:
			fmt.Println(msg1)
		case msg2 := <-c2:
			fmt.Println(msg2)
		}
	}
}
```

And the result is now as expected:

```bash
go run main.go 
Every 2000ms
Every 500ms
Every 500ms
Every 500ms
Every 500ms
Every 2000ms
Every 500ms
Every 500ms
^Csignal: interrupt
```

Worker pools:

```go
func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	go worker(jobs, results)

	for i := 0; i < 100; i++ {
		jobs <- i
	}
	close(jobs)

	for j := 0; j < 100; j++ {
		fmt.Println(<-results)
	}
}

func worker(jobs <-chan int, results chan<- int) {
	for n := range jobs {
		results <- fib(n)
	}
}

func fib(n int) int {
	if n <= 1 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```

The routine runs fine:

```bash
go run main.go 
0
1
1
2
3
5
8
13
21
34
55
89
144
233
377
610
987
1597
2584
4181
6765
10946
17711
28657
46368
75025
121393
196418
317811
514229
```

But it is very inefficient, we can add more workers, because the channels are used for sync:

```go
func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	go worker(jobs, results)
	go worker(jobs, results)
	go worker(jobs, results)
	go worker(jobs, results)

	for i := 0; i < 100; i++ {
		jobs <- i
	}
	close(jobs)

	for j := 0; j < 100; j++ {
		fmt.Println(<-results)
	}
}

func worker(jobs <-chan int, results chan<- int) {
	for n := range jobs {
		results <- fib(n)
	}
}

func fib(n int) int {
	if n <= 1 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```

The thing is, the above example does not guarantee that the Fibonacci number will come out in the right order! We would need something smarter for that.

