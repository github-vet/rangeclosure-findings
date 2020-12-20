# rangeloop-pointer-findings
References to range loop variables found on GitHub.

The findings in the issue tracker of this repository were gathered in order to better understand the impact of [this proposed change](https://golang.org/issue/20733).

## Where's the Code?!

This repository is "issue-only". It exists to capture the findings of [a static analysis bot](https://github.com/github-vet/bots), which reads Go repositories found on GitHub and captures potential problems around [range loop variable references](https://github.com/golang/go/wiki/CommonMistakes#using-reference-to-loop-iterator-variable).

Range-loop capture is the reason this code prints `4, 4, 4, 4,` instead of what you might expect.

```go
xs := []int{1, 2, 3, 4}
for _, x := range xs {
    go func() {
        fmt.Printf("%d", x)
    }()
}
```

But why raise issues for all these code examples?

## Range Loop Capture Considered Dangerous

Members of the Go language team have indicated a willingness to modify the behavior of range loop variable capture to make Go more intuitive. This change could be made despite the [strong backwards compatibility guarantee](https://golang.org/doc/go1compat) of Go version 1, **only if** we can ensure that the change will not result in incorrect behavior in current programs.

To make that determination, a large number of "real world" Go programs would need to be vetted. If we find that, in every case, the current compiler behavior results in an undesirable outcome (aka bugs), we can consider making [a change to the language](https://golang.org/issue/20733).

The goal of the [github-vet](https://github.com/github-vet) project is to motivate such a change by gathering static analysis results from Go code hosted in publicly available GitHub repositories, and crowd-sourcing their human analysis.

## How Can I Help?

We're glad you're here! We need the help of the Go community to examine our bot's findings to determine if they represent a bug or not.

Each instance we've gathered falls into one of three buckets (explained further below).
1. **Bug** :-1: - as written, the captured loop variable could lead to undesirable behavior.
1. **Mitigated** :+1: - the captured loop variable does not escape the block of the for loop, or if it does, it is handled in such a way that no undesirable behavior can occur.
1. **Desirable Behavior** :rocket: - the current range loop semantics are required for the correct operation of the program. Changing the behavior of the compiler [as outlined in this issue](https://golang.org/issue/20733) would make this code incorrect.

Findings can be classified by the community by marking the listed emoji reactions to each finding. A separate bot gathers the community assessment and uses it to label each instance automatically, in addition to input from experts.

### Desirable Behavior

We don't currently have any examples of desirable behavior which depends on the current semantics of range-loop captures. The goal of this project is to determine whether such an example exists in real-world code.

We don't think such an example does exist. This means that any examples marked as 'Desirable' will be faced with a high degree of scrutiny. Don't let this discourage you from reporting it. Just be aware that we may disagree, so is important to engage with a sense of professional detachment. Be humble and keep learning.

### Bugs

An example is a bug when the captured loop variable causes "undesirable" or "confusing" behavior. These terms are somewhat subjective as they depend on the programmer's intent. While we can't _know_ the programmer's original intent, we _can_ ask whether the actual behavior would be better achieved in a different way.

For example, the below finding is an example of undesirable behavior. A reference to the loop variable `action` is captured in the goroutine started within the block of the for loop. There is a race-condition between updating the value of `action` at the beginning of the loop and reading the value of `action` within the goroutine. If `len(d.workers) > 1`, it is usually the case that the only action to be run will be the last entry visited in the range loop. We mark this as a bug because if that were the intent, it would have been written differently.
```go
for queueName, action := range d.workers {
	q := d.queue.Job[queueName]
	go func() {
		for {
			action(q)
		}
	}()
}
```

As another example, in the below finding, undesirable behavior also occurs. Updates to the value of `n` race against the start of the goroutine, ensuring there is no guarantee that `.Run()` will be called on every value in `g.nodes`, which is clearly the intent of the range loop.
```go
for _, n := range g.nodes {
  wg.Add(1)
  go func() {
    err := n.Run()
    if err != nil {
      c <- err
    }
  }()
}
```

### Mitigated

These examples aren't bugs because they have been written in such a way that the capture is mitigated. "Mitigated" examples aren't as interesting as desirable behavior, but we want to mark them separately, so we can consider whether the static analysis could detect them automatically.

##### Examples
For instance, the below finding is 'mitigated' by explicitly passing the `reader` variable into the function started via a goroutine.
```go
for _, reader := range dec.rs {
	wg.Add(1)
	go func(r io.Reader) {
		defer wg.Done()
		tris, err := dec.newDecoderFunc(r).Decode()
		select {
		case results <- &result{tris: tris, err: err, reader: r}:
		case <-done:
			return
		}
	}(reader)
}
```
Another common instance of mitigated use is found in tests like the below finding. The `done` channel in this example is used to ensure that the goroutine finishes before the value of `tt` is updated. This means that the value of `tt` is not overwritten before the goroutine completes, so there is no race condition.

```go
for i, tt := range tests {
	txn := s.Write()
	done := make(chan struct{}, 1)
	go func() {
		tt()
		done <- struct{}{}
	}()
	select {
	case <-done:
		t.Fatalf("#%d: operation failed to be blocked", i)
	case <-time.After(10 * time.Millisecond):
	}

	txn.End()
	select {
	case <-done:
	case <-time.After(10 * time.Second):
		testutil.FatalStack(t, fmt.Sprintf("#%d: operation failed to be unblocked", i))
	}
}
```

## Marking Findings

To mark a finding, simply add the appropriate emoji reaction to the issue.
![Example demonstrating UI usage.](https://github.com/github-vet/rangeclosure-findings/blob/355212562c9eb26da4d2267ad13817e29857303d/example.png)

## Classes of Findings

VetBot does a lot of work to try and prove that no reference to a range-loop variable is improperly used. To do so, it checks the broad classes of errors seen below (along with a minimal example).

### Range Variable used in Defer or Goroutine
```
for _, i := range []int{1,2,3} {
   go func() {
     fmt.Print(i) // prints 3,3,3 (usually)
   }()
}
```
Reported as `range-loop variable i used in defer or goroutine at line 3`.

### Range Variable Reassigned in Loop Body
```
var x *int
for _, i := range []int{1,2,3} {
   x = &i
}
```
Reported as `reference to i is reassigned at line 3`.

### Range Variable Passed as Argument to a Potentially Unsafe Function

#### Type I: Function Writes Pointer to Memory
```
var intPtr *int
func unsafe(x *int) {
  intPtr = x
}

for _, i := range []int{1,2,3} {
   unsafe(&i)
}
```
Reported as `function call at line 7 may store a reference to i`.

A Type I error would also be reported in case `unsafe` uses the value of `x` inside a composite literal (i.e. `Foo{1, "bar", &x}`).

#### Type II: Function Starts a Goroutine
```
func unsafe(x *int) {
  go func() {
    fmt.Println(x)
  }()
}

for _, i := range []int{1,2,3} {
   unsafe(&i)
}
```

Reported as `function call which takes a reference to i at line 8 may start a goroutine`.

At the time of this writing, a Type II error would be reported whether `unsafe` uses the value of `x` inside the goroutine or not.

Both the Type I and II errors reported above are traced inductively throughout the call graph. Roughly speaking, if a function `f` calls a function `g` which is marked as unsafe, then `f` is also considered unsafe.

For more information on the static analysis used to produce these findings, see [the documentation for the VetBot](https://github.com/github-vet/bots/tree/main/cmd/vet-bot)
