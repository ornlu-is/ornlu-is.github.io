# Using Go build tags for defining sets of tests


Some time ago, I was looking for a way to define a clear separation in a Go project between my unit tests and my integration tests. At the time, the solution I came up with involved creating a submodule and some Makefile shenanigans to separate these into two sets. I was unhappy with this approach, since it ended up being convoluted and prone to error as the code base evolved. However, today I recently learned about Go's build tags that seemlessly allow me to separate my integration tests from my unit tests.

## What are build tags in Go?

Build tags are used by the Go compiler to know when and which code needs to be compiled. While some people use this when developing software that might have features enabled for a given operating system, *e.g.*, Windows 10 vs. some Linux distro, but have not yet been implemented into other OSs. To add a build tag to a specific `.go` file, all you have to do is add the following magic comment to the top of the file:

```go
// +build <your tag goes here>

package something
```

Note that an empty line was added after the build tag. This is not by accident, it is by design: if you forget to add the newline after the build tag, the Go compiler will interpret it as a comment instead, rendering your build tag useless. 

If you specified a build tag, say `tagus`, on some `.go` file, and you then run `go build`, that tagged file will not be compiled, build tags exclude your code from being compiled as their default behaviour. To include tagged files when compiling you simply have to add the `-tags` flag followed by the tags to add, as shown below:

```plaintext
go build -tags tagus
```

Files without any built tag are compiled irrespective of what tags you specify to the `go build` command.

## Build tag boolean logic

You probably noticed that the flag we just used was a plural noun, and you are likely thinking that this must mean we can specify several tags when compiling. You are very much correct. Moreover, build tags in Go allow for, albeit limited, boolean logic. Three boolean operations are allowed with build tags:

* OR - by separating tags with a space: `// +build one two`
* AND - by separating tags with a comma: `// +build one,two`
* NOT - by prefixing tags with an exclamation mark: `// +build !one`

## How can these be used for testing?

When you run Go tests using `go test`, you are actually compiling the code in your `_test.go` files, as well as any code used, directly or indirectly, by these files. Then the resulting binary is ran, your test output is produced, and then the binary is deleted (although there is some caching going on to build the test binary faster the next time you run `go test`).

If it compiles, then surely it can also make use of build tags. In the beginning of this blog post, I said I used this to separate unit from integration tests. The first question you might want to ask is why I want to do that. The main reason is because unit tests are fast, but integration tests tend to be very slow. In my case, my unit tests only need to build the test binary, while my integration tests require a Docker Compose stack to be built and started, which is a bit cumbersome when you are writing code that can be easily covered by unit tests.

As such, on my `integration_test.go` file, I added a simple build tag, via `// +build integration`, and then I added the following two Makefile targets:

```Makefile
integration-tests:
    go test -tags integration -v ./...

unit-tests:
    go test -v ./...
```

This means that, to only run unit tests, `make unit-tests` is the way to go. However, if I want to run all of my tests, including integration tests, I can simply call `make integration-tests` and go grab a cup of coffee because my integration tests stack takes forever to run.

