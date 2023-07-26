# Detecting Vulnerabilities in Go Code


When writing software, one might, accidentally (or not), ship the software with vulnerabilities, which are broadly defined as flaws or weaknesses in code that can be exploited by an attacker. We do not want to have those in our Go code so we need some way of minimizing the number of vulnerabilities our code has. Fortunately there are tools built by the Go community and team that can be leveraged for this.

## CWEs and CVEs

Before introducing Go tools to make your code more secure, it is important to clarify two concepts that are commonly used in the infosec industry: CWE and CVE. The first stands for Common Weakness Enumeration and refers to a catalog of weaknesses in software components. It is important to keep in mind that this does not refer to specific instances of vulnerabilities, but rather types of weaknesses that are usually found in software, hardware or firmware. The second term stands for Common Vulnerabilities and Exposures, and is a standard for identifying and distributing information on vulnerabilities on specific instances of products or systems.

## `gosec`

A popular Go tool that is sometimes ran as a linter, `gosec` scans your code's abstract syntax tree for security problems. Since this works by examining an AST and is programmed as a set of rules, it is limited in its detection scope. In fact, at the time of writing this article, it can only detect 34 issues. Moreover, each of these possible detections is mapped into a CWE, as described in `gosec`'s GitHub repository.

## `govulncheck`

Created by the Go security team, `govulncheck` is a bit more sophisticated than `gosec`. The Go security team gather data on known CVEs from multiple sources, puts these through a curation process, and makes this information publicly available. Moveover, this team also built the `govulncheck` tool, which allows us to check for these known vulnerabilities via source code inspection. However, if we really want to, it can also analyse binaries, but at the expense of information on call stacks.

## Why you should run both these tools

`gosec` looks for known CWEs, which may or may not result in CVEs, while `govulncheck` looks for known CVEs, making this pair a powerful stack to improve the security of you code. Let us craft an example where we can see these tools in action and how they differ. Create a new Go project, with version 1.19 (this is important, otherwise you will not be able to reproduce this), and place the following code in your `main.go` file:

```go
package main

import (
	"fmt"
	"math/rand"

	"golang.org/x/text/unicode/norm"
)

func main() {
	word := "cão" // "dog" in portuguese

	nfc := norm.NFC.String(word)
	nfd := norm.NFD.String(word)

	fmt.Printf("NFC/NFD: %s/%s\n", nfc, nfd)
	fmt.Printf("This is a random number: %f", rand.Float64())
}
```

This will be our test subject. At a glance, there seems to be nothing wrong with our code, but let us see what `gosec` and `govulncheck` have to say about that. After following the documentation and installing these tools, they are fairly easy to run. Let us start by looking for CWEs with `gosec`, which can be done via the following command:

```bash
gosec ./...
```

When we run this, we get the following output:

```plaintext
[gosec] 2023/07/26 23:44:50 Including rules: default
[gosec] 2023/07/26 23:44:50 Excluding rules: default
[gosec] 2023/07/26 23:44:50 Import directory: /home/luis/Projects/go_vulnerabilities_example
[gosec] 2023/07/26 23:44:51 Checking package: main
[gosec] 2023/07/26 23:44:51 Checking file: /home/luis/Projects/go_vulnerabilities_example/main.go
Results:


[/home/luis/Projects/go_vulnerabilities_example/main.go:17] - G404 (CWE-338): Use of weak random number generator (math/rand instead of crypto/rand) (Confidence: MEDIUM, Severity: HIGH)
    16: 	fmt.Printf("NFC/NFD: %s/%s\n", nfc, nfd)
  > 17: 	fmt.Printf("This is a random number: %f", rand.Float64())
    18: }



Summary:
  Gosec  : 2.16.0
  Files  : 1
  Lines  : 18
  Nosec  : 0
  Issues : 1
```

This tool immediately flagged the piece of code we were using to display a random number. Of course, for what the code does, this is a false positive result, but that isn't always the case (and this is for illustrative purposes only). We can just as easily run `govulncheck` with:

```bash
govulncheck ./...
```

Which, in turn, outputs the following report:

```plaintext
Using go1.19.1 and govulncheck@v1.0.0 with vulnerability data from https://vuln.go.dev (last modified 2023-07-24 16:24:24 +0000 UTC).

Scanning your code and 44 packages across 1 dependent module for known vulnerabilities...

Vulnerability #1: GO-2023-1840
    Unsafe behavior in setuid/setgid binaries in runtime
  More info: https://pkg.go.dev/vuln/GO-2023-1840
  Standard library
    Found in: runtime@go1.19.1
    Fixed in: runtime@go1.20.5
    Example traces found:
      #1: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.Callers
      #2: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.CallersFrames
      #3: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.Frames.Next
      #4: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.GOMAXPROCS
      #5: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.KeepAlive
      #6: main.go:4:2: go_vulnerabilities_example.init calls fmt.init, which eventually calls runtime.SetFinalizer
      #7: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.TypeAssertionError.Error
      #8: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.defaultMemProfileRate
      #9: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.efaceOf
      #10: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.findfunc
      #11: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.float64frombits
      #12: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.forcegchelper
      #13: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.funcMaxSPDelta
      #14: main.go:16:12: go_vulnerabilities_example.main calls fmt.Printf, which eventually calls runtime.plainError.Error
      #15: main.go:5:2: go_vulnerabilities_example.init calls rand.init, which eventually calls runtime.throw

=== Informational ===

Found 2 vulnerabilities in packages that you import, but there are no call
stacks leading to the use of these vulnerabilities. You may not need to
take any action. See https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck
for details.

Vulnerability #1: GO-2022-1143
    Restricted file access on Windows in os and net/http
  More info: https://pkg.go.dev/vuln/GO-2022-1143
  Standard library
    Found in: os@go1.19.1
    Fixed in: os@go1.19.4
    Platforms: windows

Vulnerability #2: GO-2022-1095
    Unsanitized NUL in environment variables on Windows in syscall and os/exec
  More info: https://pkg.go.dev/vuln/GO-2022-1095
  Standard library
    Found in: syscall@go1.19.1
    Fixed in: syscall@go1.19.3
    Platforms: windows

Your code is affected by 1 vulnerability from the Go standard library.

Share feedback at https://go.dev/s/govulncheck-feedback.
```

Not only does it inform us on a vulnerability our code has, but it also informs us on vulnerabilities that our code might end up having due the existence of known CVEs in functions of the packages we are importing but not reached by our call stack. In this case, we can also see when the vulnerability our code has was fixed, which was in Go v1.20.5.

While we are running these tools out-of-the-box in our terminal, note that they also support several output formats that are more easily processed by other programs and also have amazing integrations with popular IDEs and text editors, so you can ran these tools every time you save a file instead of waiting for some CI build to perform these detections for you, which is pretty neat.

{{< admonition type=tip title="Link to the example code" open=true >}}
https://github.com/ornlu-is/go_vulnerabilities_example
{{< /admonition >}}

## References

* `gosec` - https://github.com/securego/gosec
* `govulncheck` - https://go.googlesource.com/vuln

