# A pre-commit git hook for running Go unit tests


I mostly code in Go, which comes with the handy `go` tool. This tool has a bunch of functionalities with one of the most important (at least for me) being the ability to run tests. I love writing tests for my code because I hate being paged when I am on call. However, whenever I open a PR, sometimes I forget to run the tests locally before pushing my code and then my code ends up failing the CI builds, which overall results in a slower development process. Fortunately, I'm in the business of automation, and there is one particular tool that I can leverage so that I do not have to remember to run the tests every time: `git`` hooks.

## `git` hooks

`git` hooks is just a fancy name for scripts that get executed when certain `git` events occur. It really is that simple. However, it is important to take a while to understand when these scripts are triggered. We will focus our attention on local hooks, which are ones that run on `git` events that happen in our machine and not in the remote repository. There are four hooks that allow us to have scripts being executed at each step of a commit's lifecycle:
* `pre-commit` - after running `git commit` but before being prompted for a commit message;
* `prepare-commit-msg` - after the previous hook and is used to populated the commit message;
* `commit-msg` - after the commit message is entered;
* `post-commit` - immediately after the previous hook, but has the downside of not being able to change the outcome of `git commit`.

Out of all of these options, we want one that will block the `git commit` command if any of the tests fails, which immediately excludes the last hook. And, to be honest, it is not necessary to have a the hook run after entering the commit message because if then the tests fail, I'll have to fix them and enter the commit message once more. Since there is no need to populate the commit message with anything special, I'll be going with a `pre-commit` hook.

## A Go example

I need an example project to show this in action. So I came up with the following `main.go` file:

```go
package main

import "fmt"

func isEven(num int) bool {
	if num%2 == 0 {
		return true
	}
	return false
}

func main() {
	fmt.Println("1 is even?", isEven(1))
	fmt.Println("32 is even?", isEven(32))
}
```

As you can see, this Go application simply prints if 1 and 32 are even numbers. But it has an `isEven` function, for which we have written the following unit tests in the `main_test.go` file:

```go
package main

import "testing"

func TestIsEven(t *testing.T) {
	for _, tc := range []struct {
		name           string
		givenNumber    int
		expectedResult bool
	}{
		{
			name:           "given an even number returns true",
			givenNumber:    666,
			expectedResult: true,
		},
		{
			name:           "given an odd number returns false",
			givenNumber:    333,
			expectedResult: false,
		},
	} {
		t.Run(tc.name, func(t *testing.T) {
			res := isEven(tc.givenNumber)

			if res != tc.expectedResult {
				t.Errorf("expected result %t, but got %t", tc.expectedResult, res)
			}
		})
	}
}
```

This is just an illustrative example, hence why the Go application is so uninteresting.

## Implementing a Python `git` hook

We can can choose from several scripting languages to implement our hook. For this example, I chose to implement it in Python, for no particular reason. For that matter, I created a directory `hooks/` and a file within it called `pre-commit` with the following code:

```python
#!/usr/bin/python3

import sys
import subprocess

process_output = subprocess.run(["go", "test", "github.com/ornlu-is/go_git_hooks_example"], text=True, capture_output=True)
print(process_output.stdout)
sys.exit(process_output.returncode)
```

If you are not familiar with this, the first line of the script tells our computer which interpreter the script has to be ran with. In this case, it is Python 3, so if you do not have Python 3 installed, this Git hook will fail. Keep in mind that `git` hooks fails for any non-zero exit code, hence the last line of the script. 

## Adding Git hooks to our Go program

But how do we actually run this? If you've explored the `.git/` directory before, you probably know that it contains a `hooks/` directory where, you've guessed it, `git` hooks usually live. But this makes it very hard to share hooks with your team members, since the contents of the `.git` folder are not added to the remote repository.

My hooks are in the `./hooks/` directory, so I have to somehow tell `git` that this is where they are. Most recommendations for this paradigm revolve around using creating symlinks, but there is one very neat (and simple) one-liner that completely solves our issue:

```
git config core.hooksPath hooks/
```

This line basically rewires `git` so that it looks for hooks in the `./hooks` directory instead of in the `./.git/hooks/` directory! Moreover, this only affects the repository you are working on, meaning that if you have some different behaviour for another repository, it will be preserved. There is only one last step: our need to be executable, meaning that we just need to run:

```
chmod +x hooks/pre-commit
```

And we now have a functioning `pre-commit` `git` hook that will run our Go tests for us whenever we use the `git commit` command!

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_git_hooks_example
{{< /admonition >}}

