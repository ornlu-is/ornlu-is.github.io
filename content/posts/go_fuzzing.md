---
title: "Fuzzing with Go"
date: 2023-07-23T22:31:58+01:00
categories: ["Golang"]
draft: true
---

This tutorial introduces the basics of fuzzing in Go. With fuzzing, random data is run against your test in an attempt to find vulnerabilities or crash-causing inputs. Some examples of vulnerabilities that can be found by fuzzing are SQL injection, buffer overflow, denial of service and cross-site scripting attacks.

Fuzz testing or fuzzing is an automated software testing method that injects invalid, malformed, or unexpected inputs into a system to reveal software defects and vulnerabilities. A fuzzing tool injects these inputs into the system and then monitors for exceptions such as crashes or information leakage. Put more simply, fuzzing introduces unexpected inputs into a system and watches to see if the system has any negative reactions to the inputs that indicate security, performance, or quality gaps or issues.

When fuzzing, you can’t predict the expected output, since you don’t have control over the inputs.

The function begins with FuzzXxx instead of TestXxx, and takes *testing.F instead of *testing.T

Where you would expect to see a t.Run execution, you instead see f.Fuzz which takes a fuzz target function whose parameters are *testing.T and the types to be fuzzed. The inputs from your unit test are provided as seed corpus inputs using f.Add.

Run the fuzz test without fuzzing it, by simply running `go test`, to make sure the seed inputs pass.

Run FuzzReverse with fuzzing, to see if any randomly generated string inputs will cause a failure. This is executed using go test with a new flag, -fuzz, set to the parameter Fuzz. Copy the follwoing command: `go test -fuzz=Fuzz`.

Another useful flag is `-fuzztime`, which restricts the time fuzzing takes. For example, specifying `-fuzztime 10s` in the test below would mean that, as long as no failures occurred earlier, the test would exit by default after 10 seconds had elapsed. 

A failure occurred while fuzzing, and the input that caused the problem is written to a seed corpus file that will be run the next time go test is called, even without the -fuzz flag. To view the input that caused the failure, open the corpus file written to the testdata/fuzz/FuzzReverse directory in a text editor. 

Run `go test` again without the `-fuzz` flag; the new failing seed corpus entry will be used.

The current Reverse function reverses the string byte-by-byte, and therein lies our problem. In order to preserve the UTF-8-encoded runes of the original string, we must instead reverse the string rune-by-rune.

To examine why the input is causing Reverse to produce an invalid string when reversed, you can inspect the number of runes in the reversed string.

The entire seed corpus used strings in which every character was a single byte. However, characters such as chinese characters can require several bytes. Thus, reversing the string byte-by-byte will invalidate multi-byte characters.

We can see that the string is different from the original after being reversed twice. This time the input itself is invalid unicode. How is this possible if we’re fuzzing with strings?

The fuzzing arguments can only be the following types:
* string, []byte
* int, int8, int16, int32/rune, int64
* uint, uint8/byte, uint16, uint32, uint64
* float32, float64
* bool

Below are suggestions that will help you get the most out of fuzzing:
* Fuzz targets should be fast and deterministic so the fuzzing engine can work efficiently, and new failures and code coverage can be easily reproduced.
* Since the fuzz target is invoked in parallel across multiple workers and in nondeterministic order, the state of a fuzz target should not persist past the end of each call, and the behavior of a fuzz target should not depend on global state.

Fuzz tests are run much like a unit test by default. Each seed corpus entry will be tested against the fuzz target, reporting any failures before exiting.

### References

* https://go.dev/security/fuzz/
* https://go.dev/doc/tutorial/fuzz
* https://pkg.go.dev/cmd/go#hdr-Testing_flags
* https://google.github.io/oss-fuzz/getting-started/new-project-guide/go-lang/
* https://google.github.io/oss-fuzz/
