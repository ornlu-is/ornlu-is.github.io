---
title: "Go Concurrency Patterns: Pipeline"
date: 2023-08-21T16:24:34+01:00
description: "How to implement the pipeline concurrency pattern in Go"
categories: ["Golang"]
draft: false
---

Yet another Go concurrency pattern! This particular pattern is extremely helpful in composing several transformations from data incoming from a stream, and is know as the pipeline concurrency pattern. In a pipeline, we define several stages, which are nothing more than objects that take data in, perform some operation on it, and then output the transformed data.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_pipeline_pattern
{{< /admonition >}}

## The Pipeline pattern in Go

{{< figure src="/images/go_pipeline_pattern/pipeline.png" width=0.5 height=0.5 title="Pipeline pattern" >}}

Translating the above definition to Go, the pipeline pattern is simply a function that takes a channel, performs some operation on the data from that channel, and outputs it to another channel. Usually, these functions are easily composable, meaning you can chain several calls that will represent exactly what is going on with your data.

For this example, I created a simple function that writes a bunch of floats to a channel via goroutine and returns that channel:

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

This will act as our data stream. Suppose that, for some reason, you need to perform two operations on the values of your stream: you need to take each number's power, and then multiply that by two. From our pipeline pattern definition, we can easily deduce what it will look like in code. Firstly, and obviously, the function will take as argument a channel and return a channel. Then, since we want to perform an operation on each value from the input channel and write it to the output channel, we need to spawn a goroutine that will handle that. And that's it! Pretty simple, right? To take the power of each input we write the following function:

```go
func power(data <-chan float64) <-chan float64 {
	ch := make(chan float64)

	go func() {
		defer close(ch)
		for value := range data {
			ch <- value * value
		}
	}()

	return ch
}
```

Likewise, to duplicate each input, our function looks very much like the previous one:

```go
func duplicate(data <-chan float64) <-chan float64 {
	ch := make(chan float64)

	go func() {
		defer close(ch)
		for value := range data {
			ch <- 2. * value
		}
	}()

	return ch
}
```

Let's put this all together in our `main` function. To see this in action, we simply have to create our data stream, feed it as input to the `power` function, which in turn is fed as input to the `duplicate` function, thus forming our pipeline: 

```go
func main() {
	dataStream := numberStream()

	for value := range duplicate(power(dataStream)) {
		fmt.Println(value)
	}

	fmt.Println("bye")
}
```

{{< admonition type=tip title="Note" open=true >}}
We can range over the output of these functions because they are built as generators. If you are curious about this design pattern, check out this other post that I've written:
* https://ornlu-is.github.io/go_design_pattern_generator/
{{< /admonition >}}

Now, if we run the above code, we get exactly what we expected, which is pretty cool:

```plaintext
2
8
18
32
50
72
98
128
162
200
bye
```
