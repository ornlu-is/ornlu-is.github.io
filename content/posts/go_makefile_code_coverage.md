---
title: "Enforcing test coverage in Go with Makefile"
date: 2023-08-17T23:22:19+01:00
description: ["How to enforce a test coverage for Go code using a Makefile"]
categories: ["Golang"]
draft: false
---

Makefiles are a popular way of making the development process easier, since they can be used to chain several commands that allow developers to build, test, run, etc. their code. Additionally, they can also be used to create a make-based build/test system. In this post, I'm going to cover something how to set up a Makefile rule to test Golang code and enforce test coverage, *i.e.*, have the rule fail if a predefined test coverage threshold is not met.

## TL;DR, just let me copy it

Let us take a look at the Makefile and then we'll walk through it step by step:

```Makefile {linenos=table}
SHELL=/bin/bash
TEST_COVERAGE_THRESHOLD=80.0

test:
    go test ./... -coverprofile=coverage.out 
    coverage=$$(go tool cover -func=coverage.out | grep total | grep -Eo '[0-9]+\.[0-9]+') ;\
    rm coverage.out ;\
    if [ $$(bc <<< "$$coverage < $(TEST_COVERAGE_THRESHOLD)") -eq 1 ]; then \
        echo "Low test coverage: $$coverage < $(TEST_COVERAGE_THRESHOLD)" ;\
        exit 1 ;\
    fi
```

## Okay, I regret just copying it, how does this work?

The first two lines are defining global variables. The `SHELL` variable is present in all Makefiles and specifies which shell we are using. By default, Makefile uses `sh` but we want more functionalities so we want our Makefile to use `bash`, so we place the path to `bash` in our `SHELL` variable. The second variable is our desired minimum test coverage, *i.e.*, the value below which our make rule should fail.

Line 4 simply specifies the rule name, and is followed by its definition. We want to keep our Makefile simple and intuitive, so we name our rule `test`, meaning that this can be ran by writing `make test`.

On line 5 we test our Go code. In case any test fails, our rule execution terminates promptly, printing the test report. `./...` tells the `go` tool to recursively look for test files in our project directory and run all files of the format `*_test.go`. Finally, the `-coverprofile=coverage.out`, outputs the test coverage profile into a file called `coverage.out`. We do not directly care about what is written in this file.

Line 6 is where we make use of the `coverage.out` file. By running `go tool cover -func=coverage.out` we now calculate the percentage of test coverage for each function as well as the total coverage of our project. We are defining a simple global threshold, so we have to extract only the line of the pertaining to the total coverage. Fortunately, since this line starts with `total`, we can simply pipe the result from `go tool cover -func=coverage.out` into `grep` and tell it to extract the line containing that string using `grep total`. However, this line still needs a bit more processing since we have to remove everything except for the percentage of total coverage. In comes `grep` once again, we just pipe the result of the previous `grep` command to `grep -Eo '[0-9]+\.[0-9]+'`, where the `E` flag tells `grep` that our pattern is a regular expression, and the `o` flag is to guarantee we fetch only nonempty parts of the result. Finally, we save all of this in a local `coverage` variable.

Since we no longer have any use for our `coverage.out` file, we can safely delete it in line 7 using `rm coverage.out`.

On Line 8, we check if the our test coverage is lower than our defined threshold. Both the obtained test coverage and the test coverage threshold are given as floating point numbers, which means that they cannot be natively compared when using `bash`. However, we can use `bc` for this purpose. `bc` stands for basic calculator and, if given a condition, it returns 1 if that confition is true and 0 otherwise. Which means that all that is left is check if the output we obtain from `bc` is equal to 1, which is achieved through `-eq 1`. 

On lines 9 and 10, we print an informative message that the test coverage is too low and terminate the rule execution with error code 1. On the last line, we close our `if` block.

It took me more time than I care to admit to find out how to do this, so I am writting it down so I do not forget it.