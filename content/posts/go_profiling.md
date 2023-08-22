---
title: "Profiling Go"
date: 2023-08-22T08:39:28+01:00
categories: ["Golang"]
draft: true
---

## Some Matt's lectures

Using pprof to finding leaking goroutines (what does this mean) and looking at how we can tweak CPU performance and see that in the profiling.

The number one way you leak memory in Go is to leak a goroutine and one of the two ways you can leak a goroutine is to leak a socket. This might happen when you do not close the body of a request, causing the socket to hung up.

There's two ways to find out you have a goroutine leak. One is, you run your program, and eventually it dies because it runs out of memory. The other way is, you build your program, and as part of your testing you run some traffic and you look at tools like pprof to see how is my program behaving. Did the number of goroutines just keep going up?

Skip from pprof, and go to Prometheus.

Build simple http server that fetches stuff from http://jsonplaceholder.typicode.com/

Use prometheus to export metrics to the metrics endpoint

In Linux, if goroutines keep going up, and so do open fds (file descriptors), then this is almost surely an issue with unclosed sockets (leaking sockets). If the open fds are not going up, then it's a goroutine issue.

## From GopherCon 2019

Three different types of profiling and improving the performance of two different programs.

Use the Moby Dick text from Gutenberg library.

Use `ls` to find the size of the document.

Use `time` to check how long the program takes. Use the program to count the words. Compare with `wc` to get a baseline. They will get different answers on the number of words, but that doesn't really matter for us.

The program is much slower and that is what matters.

First thing to do is profile the program. Use `runtime.pprof`.

It goes slower with profiling. `pprof` gets the OS to set up an interrupt on a very frequent timer (100x/sec), which adds a bit of overhead. But we benefit from having a profile.

There are three distinct starts of trees we see in pprof. There are three types of proeminent call stacks. Every time we stop the process we sample the current call stacks of the active goroutines and then when we get to the end of the program, the ones that show up the most are obviously the ones that were doing the most work.

`syscall` shows up as the thing that uses the most (all) time in our program. Why? Too many syscalls. But why are there too many syscalls? because in Go, all our read and write operations are unbuffered by default. If you want buffering, you should buffer stuff yourself.

What is the other stuff that shows up in pprof? It is quite expensive to sleep one thread and wake up another. And one of the reasons why this can happen, you notice that in syscall.Read, dispatches to generic syscall.syscall. Goruntine has a timer and if that timer elapses before the syscall is returned it's time to panic and make a new thread to make up the pool of threads that service the goroutines. If the syscall does return fast enough, that's okay.

We spend so much time in syscall that the goruntime has to replace that thread which went off to do a syscall, with another one to make up the difference. Every time we come back from syscall now there is a thread that should be in the scheduler and an extra one that is just hanging around.

This is the source of the overhead in our program.

Fix this by buffering in the code.

This brings down the performance by one order of magnitude.

Looking at the CPU profile again, our program just isn't visible in the profile (because the times are super small). Profiling is triggered in a periodic fashion, which means that if our program doesn't run long enough, we're not gonna make many samples. So the CPU does not tell us anything useful, mostly just background noise.

But sometimes, it points to mallocgc. So maybe memory usage is the key to the difference between what our program is doing and what wc is doing.

This gives us a hint that maybe we should try a different kind of profiling of our program. So let us switch from CPU profiling to memory profiling.

Rather than capturing CPU cycles and using an interrupt from the OS. The Go runtime itself is going to catch the stack trace that lead to that allocation.

Different types of profiling have different overheads.

All allocations come from main.readbyte.

To profile every allocation, we just have to mess with some settings from the profiler, i.e., set the sampling rate to 1.

But this will help us see very allocation that happened in our program.

It's leaking because we declare and array, slice that array into a slice, and pass it to read. io.Reader is an interface, the runtime and the compiler don't know at runtime, what reader is actually gonna be passed. This forces this allocation to escape to the heap and that's where all memory allocations are being recorded.

We can just reuse the buffer and reslice it over and over again.

We reduced memory consumption so much that things that would previously never show because they were negligible, will now start showing up.

Follow the giant arrow because that tells us where most of the time is being spent.

Trace tool is another thing we can explore. It builds a trace profile. But has its own tool. We can identify if we could use multiple processors to split computation.

User is how many CPU seconds we used. If this increases but real stays the same, it means that we are using more CPUs, but we are not really computing things faster. The size of a trace is based on the number of goroutines and their interactions.

Sometimes, the amount of time it takes to complete a task that is handed off to a goroutine is larger than the overhead of starting and managing the goroutine. 

## Understanding the Stack and the Heap

There's two kinds of memory in your program.

There's stacks. In Go there are multiple stacks, one for each goroutine. And then we have heap memory, which is everything else.

How do I know, in my code, if my variable lives on the stack or on the heap? The short answer is: you don't. But does this matter?
* As far as correctness of the program, it does not matter. Go takes care of that, assigning variables to the appropriate stack or the heap.
* But it does affect performance of the program, so you might need to know.
* Anything on the heap is managed by the garbage collector.
* Which is very good but does cause some latency. And it may cause some latency for your whole program, not just the part creating garbage. So for some programs you may be concerned about how much garbage you are putting on the heap.

Only matters if:
* Your program is not fast enough. If it is fast enough, you are already done;
* Your program is not fast enough and you have benchmarks to prove it. No guessing, data is your friend.
* And if those benchmarks show excessive heap allocations.

Only maybe you should look at it.

Optimize for correctness first, not performance.

Sharing down typically stays on the stack. Sharing down refers to passing pointers, passing references to things.

Sometimes the compiler knows that it is not safe to leave certain bits of memory in a given stack frame. So instead, it is saved somewhere on the heap. This is called "escaping to the heap". It does not get moved at runtime, this is a compile time thing.

Sharing up typically escapes to the heap. This refers to returning pointers, returning references. Returning things that have pointers in them.

The word choice is deliberate: typically. It is not always true. Only the compiler truly knows where the memory is allocated.

When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors.

By "cannot prove" we are referring to "Escape analysis".

In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

If only the compiler knows, we can ask the compiler.

`go help build` then check the option `-gcflags`. Look at `go tool compile -h`. It has the `-m` option which prints out the optimization decisions, which is how we can ask the compiler what it is doing. `go build -gcflags "-m"`. Using `-m=2` yields more verbose output.

When are values constructed on the heap? 
* When a value could possibly be referenced after the function that constructed the value returns. It's not enough to just say "variables on the stack", you've got to remember that variables don't just go on the stack, they go in the stack frame for a particular function. So if a particular function has already returned, its stack frame isn't being used anymore, but if the variable is gonna be used after that happens. Then it has to go to the heap. Go takes a safe approach, where it moves the variable to the heap if there is the slightest possibility that the variable is referenced.
* When the compiler determines a value is too large to fit on the stack.
* When the compiler doesn't know the size of a value at compile time. For example, if you have a slice and its size is going to be decided at runtime, there is no way for the compiler to know if that slice is going to be small enough to be in the stack or not.

Commonly allocated values to the heap:
* values shared with pointers
* variables stored in interface variables
* anonymous function variables, and any variables captured by a closure
* backing data structures for maps, channels, slices and strings

This explains why the io.Reader interface is the way it is. In other words, why doesn't it return the bytes? Why do you have to give it the bytes?



### References

* https://go.dev/blog/pprof
* https://github.com/google/pprof