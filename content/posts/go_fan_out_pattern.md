---
title: "Go Concurrency Patterns: Fan-Out"
date: 2023-08-21T14:40:12+01:00
description: "How to implement the fan-out concurrency pattern in Go"
categories: ["Golang"]
draft: false
---

I have written a blog post about the fan-in concurrency pattern and, unlike most texts on this matter, I left its counterpart, the fan-out concurrency pattern, to have its own post. While these two patterns are mostly used in tandem, I believe that it is fundamental to understand them separately, so as to not create any mental blockers that would coherce us to only use one pattern when the other is also required.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_fan_out_pattern
{{< /admonition >}}

## The Fan-Out pattern in Go

{{< figure src="/images/go_fan_out_pattern/fan_out.png" width=0.5 height=0.5 title="Fan-Out pattern" >}}

While the fan-in concurrency pattern equates to multiplexing, *i.e.*, combining several data streams into a single one, the fan-out pattern does the opposite, it takes a single data stream and creates several concurrent streams. The obvious application for this pattern is when you want to concurrently process a data stream and you do not need your data to be processed in order. This last observation is crucial, since this pattern does not preserve the order by which the data is handled.

As with the fan-in example, I'll be creating a data stream from strings with the names of numbers, like so:
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
```

Demultiplexing this is incredibly straightforward. All we need is to read from the original channel into as many new channels as we'd like. As such, we'll create a function, `demultiplexer`, which will spawn a new goroutine that will read from the original channel. Additionally, it will return a channel just so that we have some signal of when the goroutine stopped reading from the original stream. We end up with something like this:

```go
func demultiplexer(worker string, ch <-chan string) chan struct{} {
	stop := make(chan struct{})

	go func() {
		defer close(stop)
		for v := range ch {
			fmt.Println(worker, v)
		}
	}()

	return stop
}
```

Now, if we want to split our original stream into two streams, it suffices to call `demultiplexer` twice. We can then read from these new streams with a `for-select` loop, as shown below:

```go
func main() {
	originalStream := someNumberStrings()

	demuxedStream1 := demultiplexer("demux1", originalStream)
	demuxedStream2 := demultiplexer("demux2", originalStream)

	for {
		if demuxedStream1 == nil && demuxedStream2 == nil {
			break
		}

		select {
		case _, ok := <-demuxedStream1:
			if !ok {
				demuxedStream1 = nil
			}
		case _, ok := <-demuxedStream2:
			if !ok {
				demuxedStream2 = nil
			}
		}
	}

	fmt.Println("bye")
}
```

Note that, due to the nature of goroutines, whenever you run this program, you'll get a different result. For example, I obtained the following:

```plaintext
demux2 one
demux2 two
demux2 three
demux2 four
demux2 five
demux1 six
demux1 eight
demux1 nine
demux1 ten
demux2 seven
bye
```

But you can, and most certainly will, obtain a different one. This is inline with what I stated at the beginning of this post: this pattern is only useful if we do not care about the order in which our data is handled.
