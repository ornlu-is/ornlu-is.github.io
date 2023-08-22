# Why does Go's io.Reader interface take a slice of bytes as argument?


I rencetly watched a GopherCon talk titled "Understanding Allocations: The Stack and the Heap" by Jacob Walker, and found it really interesting, especially the final conclusion on why the `io.Reader` interface is the way it is. As it turns out, it is related to how memory allocation works in Go. More specifically, where the memory is allocated.

## The `io.Reader` interface

There isn't much to say here, everyone that has been programming in Go has surely found this interface out in the wild multiple times, and it looks like this:

```go
type Reader interface {
    Read(b []byte) (n int, err error)
}
```

But why does it return the number of bytes read instead of returning a slice of bytes? It would make sense that if I tell something to read, it would just give me what it read, instead of requiring me to allocate a slice where it will write to. To understand this, we have to go on a little detour.

## Stack and Heap memory in Go

Broadly speaking, there are two places where variables can be allocated in your Go program. There is the heap, and then there are the stacks, usually one for each goroutine that exists. For the purpose of this post, the main difference between a stack and a heap, is that whatever is in the heap eventually gets garbage collected.

This prompts one immediate question: where will my variables be allocated? In essence, there is no way of knowing this while you are writing code, only the compiler can answer this. However, there are some instances where your variables are **typically** written to the heap. **Usually**, whatever is shared down, *i.e.*, passing pointers to things, stays on the stack and whatever is shared up, *i.e.*, returning pointers, references, or things with pointers in them, tends to go on the heap. Additionally, there are some more use cases where memory is **typically** allocated on the heap:

* When a value could possibly be referenced after the function that constructed the value returns;
* When the value is too large to fit on the stack;
* When the size of a value is unknown at compile time;
* When values are shared with pointers;
* When variables are stored in interface variables;
* When variables store anonymous functions, or are variables captured by a closure.

Again, this is **usually**, only the compiler really knows where stuff gets allocated.

## A worked example

Let us look at a practical example. Below I have a simple piece of code, with a `main` function that calls the function `read`, assigns its return value to a variable, and then prints it out using `println` (using `fmt.Println` will yield different results). The `read` function creates a slice of two bytes, assigns a value to each of those, and then returns it.

```go
package main

func read() []byte {
	b := make([]byte, 2)
	b[0] = 1
	b[1] = 2
	return b
}

func main() {
	b := read()
	println(b)
}
```

So what is going on the heap and what is staying in the stack? We can investigate this by passing some flags to our `go build` command. When we use the `go build` tool, the underlying tool is the `compile` tool. And since we are looking for a flag that allows us to investigate what optimisation decisions are being performed by the compiler, we look for those flags using `go tool compile -h`, which uncovers the following suspect:

```plaintext
-m	print optimization decisions
```

Nice, we've found what we were looking for. Running `go build -gcflags "-m"` in our directory gives us the following output:

```plaintext
./main.go:3:6: can inline read
./main.go:10:6: can inline main
./main.go:11:11: inlining call to read
./main.go:4:11: make([]byte, 2) escapes to heap
./main.go:11:11: make([]byte, 2) does not escape
```

From the fourth of the output, we can see that, in line 4 of our code, the variable allocated has been allocated to the heap. Why? 

On line 4, we are creating a slice of bytes inside the function `read`. A slice is essentially a pointer to an underlying array, and two integers, thus, when the function `read` returns and assigns its return value to `b`, we now have two pointers to the same memory position. However, the pointer created inside `read` cannot be used by anything in our code, it is unreachable. As such, it is garbage and must be garbage collected. Hence, it goes on the heap.

So what happens if, instead of having our `read` function create and return a slice of bytes, we have it receive a slice of bytes as argument and populate it with data? For that purpose, I wrote the following program:

```go
package main

func read(b []byte) {
	b[0] = 1
	b[1] = 2
}

func main() {
	b := make([]byte, 2)
	read(b)
	println(b)
}
```

For which we have to following output of the optimisation decisions:

```plaintext
./main.go:3:6: can inline read
./main.go:8:6: can inline main
./main.go:10:6: inlining call to read
./main.go:3:11: b does not escape
./main.go:9:11: make([]byte, 2) does not escape
```

In line 3, `b` does not escape because it is a copy of the slice defined in the `main` function, which means that it is a copy of the pointer (and two integers for the slice capacity and length), not of the underlying array, that allocation has already been performed in `main`.  

## So why does `io.Reader.Read` receive a byte slice as argument?

Now we are ready to answer this question. And the answer is extremely simple: because if it returned a slice of bytes instead, a lot more garbage collection would have to be performed and this would result in increased latency.

## Final note

Please do not write your program with the mindset of minimizing how much stuff gets written into the heap. Strive for correctness and readability, and only go for this type of optimisation if you require your program to go faster and you have data to support your hypothesis that latency is being caused by garbage collection.

