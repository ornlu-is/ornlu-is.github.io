---
title: "Go Concurrency Patterns: Fan-In"
date: 2023-08-21T09:38:20+01:00
description: "How to implement the fan-in concurrency pattern in Go"
categories: ["Golang"]
draft: false
---

It is not uncommon to have a piece of software that is concurrently reading from multiple streams of data. However, for a multitude of possible reasons, we might want to aggregate these streams into a single one, for example, to send the data to another service. Fortunately, this is not a new problem, and the solution for it is well known as the Fan-In pattern. 

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_fan_in_pattern
{{< /admonition >}}

## The Fan-In pattern in Go

{{< figure src="/images/go_fan_in_pattern/fan_in.png" width=0.5 height=0.5 title="Fan-In pattern" >}}

As stated before, the idea behind this is incredibly simple: the fan-in pattern combines several data streams into one. You've possibly seen this defined in terms of "multiplexing". Fret not, that is just a fancy word for "merging multiple streams into one". There is nothing like learning by doing, so let us get right to it.

The first thing we need is to emulate multiple streams of data. For that matter, I created two functions `someNumberStrings` and `someNumbers` which both spawn a goroutine and return a channel to where the aforementioned goroutine will write some strings:

```go
func someNumberStrings() <-chan string {
	ch := make(chan string)
	numberStrings := []string{"one", "two", "three", "four", "five", "six", "seven", "eight", "nine", "ten"}

	go func() {
		for _, numberString := range numberStrings {
			ch <- numberString
		}

		close(ch)
		return
	}()

	return ch
}

func someNumbers() <-chan string {
	ch := make(chan string)
	numbers := []string{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"}

	go func() {
		for _, number := range numbers {
			ch <- number
		}

		close(ch)
		return
	}()

	return ch
}
```

Before diving into the code for multiplexing these data streams, let us think about what we need. The two data streams will be combined into one, so we need to create a new channel, which will be our multiplexed channel (or stream). Then, we'll need a function, `multiplex`, that reads from the two channels and writes to our multiplexed channel. Since we are writing concurrently, we must also read concurrently, so we'll require two goroutines calling our `multiplex` function. As usual with channels, we'll have to close the multiplexed channel once we are done reading. Finally, we need to synchronize all of this. For that purpose, we'll employ a `WaitGroup` from the `sync` package. Putting it all together yields the following:

```go
func fanIn(channels ...<-chan string) <-chan string {
    // create WaitGroup
	var wg sync.WaitGroup

    // create multiplexed stream
	multiplexedStream := make(chan string)

    // function that takes data from one of the two streams
    // and writes it into our multiplexed stream
	multiplex := func(c <-chan string) {
        // when we finish reading from the data stream, tell 
        // our WaitGroup that we are no longer reading from that stream
		defer wg.Done()

        // write from the data stream to the multiplexed stream
		for i := range c {
			multiplexedStream <- i
		}
	}

    // tell our WaitGroup that we are waiting for two stream to finish
	wg.Add(len(channels))

    // read concurrently from the data streams into the multiplexed stream
	for _, c := range channels {
		go multiplex(c)
	}

	go func() {
        // wait for both streams to close and then close the multiplexed stream
		wg.Wait()
		close(multiplexedStream)
	}()

	return multiplexedStream
}
```

Now that we are armed with all the necessary pieces, we can put this together in our `main` function:

```go
func main() {
	ch1 := someNumberStrings()
	ch2 := someNumbers()

	exit := make(chan struct{})
	mergedCh := fanIn(ch1, ch2)

	go func() {
		for val := range mergedCh {
			fmt.Println(val)
		}

		close(exit)
	}()

	<-exit

	fmt.Println("bye")
}
```

When I run this, I get the following output:

```plaintext
one
1
two
three
2
3
four
4
5
five
six
6
seven
eight
7
8
nine
ten
9
10
bye
```

Note that there is no specific ordering of the output, *i.e.*, we are not processing one input from each stream at a time. Moreover, you might get a different output depending on which computer you run this in. As is, this pattern does not care about the ordering of the data in the multiplexed stream.
