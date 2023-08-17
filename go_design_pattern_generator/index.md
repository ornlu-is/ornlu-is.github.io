# Go Design Patterns: Generator


I like the generator pattern. I hadn't realized it, but I had already encountered this pattern before when I used to program in Python. I recently found myself requiring to loop over a large sequence of numbers. Naively, I created a slice with all the values I required and then I looped over them. However, I was not satisfied with this solution so I went digging and found the generator pattern.

This is an incredibly simple pattern. It's purpose is very direct: a generator is something that yields a sequence of values one at a time. Let us look at an example.

## Without the generator pattern

Let us assume that, in our project, something that we need to do quite regularly is to range over a set of linearly spaced floating point numbers, *e.g.*, 0.1, 0.3, 0.5, etc. This is not that uncommon in scientific computing or statistics. The first time we implement this, we went with something like

```go
start := 0.1
end := 0.6
step := 0.1
for i := start; i < end; i += step {
    // do something
}
```

But then we had to implement this again in another piece of code, and another, and the boilerplate code just started piling up.

## With the generator pattern

Below is the implementation of our generator function:

```go
func linearSpaceGenerator(start, end, step float64) chan float64 {
	ch := make(chan float64)

	go func(ch chan float64) {
        defer close(ch)
		
        for n := start; n < end; n += step {
			ch <- n
		}
	}(ch)

	return ch
}
```

We create a bidirectional unbuffered channel and spawn a goroutine to write into it. Since the channel is unbuffered, every single value that we write into it will block until another goroutine reads the channel. Then, after writing all our values into the channel, we simply call the built-in `close` function to let downstream processes know that we are done.

Using this function is painfully easy and looks much more elegant:

```go
package main

import "fmt"

func linearSpaceGenerator(start, end, step float64) chan float64 {
	ch := make(chan float64)

	go func(ch chan float64) {
        defer close(ch)

		for n := start; n < end; n += step {
			ch <- n
		}
	}(ch)

	return ch
}

func main() {
	for i := range linearSpaceGenerator(0.1, 2., 0.1) {
		fmt.Printf("i: %f\n", i)
	}
}
```

So we have the goroutine spawned by `linearSpaceGenerator` sending values to a channel and in our main function we have the main goroutine (yep, it is a goroutine) reading from this channel. In essence, this is not the most necessary pattern ever, but it does look much better than constantly having to write C-style `for` loops.

