---
title: "Go Concurrency Patterns: Tee Channel"
date: 2023-08-21T19:39:39+01:00
description: "How to implement the tee channel concurrency pattern in Go"
categories: ["Golang"]
draft: false
---

If you've ever used the Linux `tee` command, you can probably guess what this pattern is about. At a first glance, this might seem similar to the fan-out concurrency pattern and, in a way, it is. But there is one crucial difference. The fan-out concurrency pattern splits the input from one channel into several channels for concurrent processing, while the tee channel pattern creates two channels with the exact same data as the original one.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_tee_channel_pattern
{{< /admonition >}}

## The Tee Channel pattern in Go

{{< figure src="/images/go_tee_channel_pattern/tee.png" width=0.5 height=0.5 title="Tee channel pattern" >}}

I think this is a really cool pattern. Not because of what it does, but because it forces us to reason about concurrency in a careful manner. First, let us set the stage. I have written a very simple generator below which we will use as our data stream:

```go
func numberStream() <-chan float64 {
	ch := make(chan float64)
	numberStrings := []float64{1., 2., 3., 4., 5., 6., 7., 8., 9., 10.}

	go func() {
		for _, numberString := range numberStrings {
			ch <- numberString
		}

		close(ch)
		return
	}()

	return ch
}
```

Now we are ready to start reasoning on how we can tee our channel into two. Fundamentally, what we want is to create a function, `teeChannel`, that takes one channel as input, and returns two channels, both with the same data in them. Based on this, we can write the following naïve implementation:

```go
func teeChannel(c <-chan float64) (<-chan float64, <-chan float64) {
	tee1 := make(chan float64)
	tee2 := make(chan float64)

	go func() {
		defer func() {
			close(tee1)
			close(tee2)
		}()

		for val := range c {
			tee1 <- val
			tee2 <- val
		}

		return
	}()

	return tee1, tee2
}
```

In our main function, we'd call this function in the following way:

```go
func main() {
	done := make(chan struct{})
	defer close(done)

	dataStream := numberStream()

	teedStream1, teedStream2 := teeChannel(dataStream)

	for val1 := range teedStream1 {
		fmt.Printf("tee1: %f\n", val1)
		fmt.Printf("tee2: %f\n", <-teedStream2)
	}
}
```

When we run this, everything seems fine! But it really isn't. Note that, in `teeChannel`, we are writing first to `tee1` and only then to `tee2`. What happens if we swap out the order of the writes and then try to run our code? We get the following error:

```plaintext
fatal error: all goroutines are asleep - deadlock!
```

So why is this happening? The reason is fairly simple. `tee2` (the same applies to `tee1`) is an unbuffered channel, meaning that, when we write to it, if there is nothing that reads from it, the goroutine will block until there is something that reads the data in this channel before continuing execution. However, we are reading from `tee1` first. So our program is stuck trying to read from `tee1` while waiting from something to read from `tee2`, hence the deadlock.

We can solve this issue by employing a `for-select` loop, coupled with setting a local copy of each channel to `nil` if it has already read a value. This will ensure that writes to one channel will not block writes to another and that both channels get written into. The end results looks like this:

```go
func teeChannel(c <-chan float64) (<-chan float64, <-chan float64) {
	tee1 := make(chan float64)
	tee2 := make(chan float64)

	go func() {
		defer func() {
			close(tee1)
			close(tee2)
		}()

		for val := range c {
			for i := 0; i < 2; i++ {
				var tee1, tee2 = tee1, tee2
				select {
				case tee1 <- val:
					tee1 = nil
				case tee2 <- val:
					tee2 = nil
				}
			}
		}

		return
	}()

	return tee1, tee2
}
```

Now we are free from deadlocks, but there is one limitation to our implementation. We are still using unbuffered channels which means that, when we tee our channel into two, have to read one value from one channel, and one value from the other. If we just keep reading from one channel, we'll end up just writing the data to that channel twice. In essence, we not only have to care about the writing process, but also the reading process when using this pattern, which makes it prone to errors.
