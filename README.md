# rangeclosure-findings
Range closure findings in GitHub projects.

The findings in the issue tracker of this repository were gathered to understand the impact of [this proposed change](https://golang.org/issue/20733).

### **Currently all findings should be considered preliminary. Stay tuned for more information.**

## Where's the Code?!

This repository is "issue-only". It exists to capture the findings of [a static analysis bot](https://github.com/github-vet/vet-bot), which reads Golang repositories found on GitHub and captures examples of the [range loop capture error](https://github.com/golang/go/wiki/CommonMistakes#using-reference-to-loop-iterator-variable).

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

Members of the Go language team have indicated a willingness to modify the behavior of range loop variable capture to make the behavior of Go more intuitive. This change could theoretically be made despite the [strong backwards compatibility guarantee](https://golang.org/doc/go1compat) of Go version 1, **only if** we can ensure that the change will not result in incorrect behavior in current programs.

To make that determination, a large number of "real world" `go` programs would need to be vetted. If we find that, in every case, the current compiler behavior results in an undesirable outcome (aka bugs), we can consider making [a change to the language](https://golang.org/issue/20733).

The goal of the [github-vet](https://github.com/github-vet) project is to motivate such a change by gathering static analysis results from Go code hosted in publicly available GitHub repositories, and crowd-sourcing their human analysis.

## How Can I Help?

We're glad you're here! We need the help of the Go community to examine our bot's findings to determine if they represent a bug or not.

Each instance falls into one of three buckets (explained further below).
1. **Bug** :-1: - as written, the captured loop variable could lead to undesirable behavior.
1. **Mitigated** :+1: - the captured loop variable does not escape the block of the for loop, no undesirable behavior can occur.
1. **Desirable Behavior** :rocket: - the current capture semantics are required for the correct operation of the program. Changing the behavior of the compiler would break this code.

Findings can be classified by the community by marking the listed emoji reactions to each finding. A separate bot will gather the community assessment and consider it in addition to the assessment of experts.

### Desirable Behavior

We don't currently have any examples of desirable behavior which depends on the current semantics of range-loop captures. The goal of the project is to determine whether such an example exists.

We don't think such an example exists. This means that any examples marked as 'Desirable' will be faced with skepticism and a high degree of scrutiny. This should not discourage reporting that they suspect desirable behavior. Let us know! We will chime in and give our view. Just be aware that we may disagree, so is important to engage with a sense of professional detachment. Be humble and keep learning.

### Bugs

An example is a bug when the captured loop variable causes "undesirable" or "confusing" behavior. These terms are somewhat subjective but it depends on the programmer's intent. While we can't _know_ the programmer's original intent, we _can_ ask whether the code exhibits a race condition and whether the actual behavior could be better achieved in a different way.

For example, [the below finding](https://github.com/yamamoto-febc/jobq/blob/e84914ddcb6230dc1ce6a825ef787c87563746b6/jobqueue.go#L120-L127) is an example of undesirable behavior. A reference to the loop variable `action` is captured in the goroutine started within the block of the for loop. There is a race-condition between updating the value of `action` at the beginning of the loop and reading the value of `action` within the goroutine. If `len(d.workers) > 1`, the only action to be run will be the last entry visited in the range loop. We mark this as a bug because if that was the intent, it would have been written differently.
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

As another example, in [the below finding](https://github.com/zikichombo/plug/blob/06941afb0420c1fbd344a7665b1a6cf2706a6981/graph.go#L37-L45), undesirable behavior also occurs. Updates to the value of `n` race against the start of the goroutine, ensuring there is no guarantee that `.Run()` will be called on every value in `g.nodes`, which is clearly the intent of the range loop.
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

These examples aren't bugs because they have been written in such a way that the capture is mitigated. "Mitigated" examples aren't as interesting as desirable behavior, but we want to mark them separately, so we can consider augmenting the analysis procedure to detect them someday.

##### Examples
For instance, the [below finding](https://github.com/wallix/triplestore/blob/4099dd913851642f2c0b71f9c1a0c6887748849c/decode.go#L263-L274) is 'mitigated' by explicitly passing the `reader` variable into the function started via a goroutine.
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
Another common instance of mitigated use is found in tests like [the below finding](https://github.com/pi-pi-miao/eye/blob/6183d469de4cf8d7fcc95e2781827db079601911/vendor/go.etcd.io/etcd/mvcc/kv_test.go#L411-L430). The `done` channel in this example is used to ensure that the goroutine started finishes before the end of the loop. This means that the value of `tt` is not overwritten before the goroutine completes, so there is no race condition.

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

### Marking Findings

To mark a finding, simply add an emoji reaction to the issue.
![Example demonstrating UI usage.](https://github.com/github-vet/rangeclosure-findings/blob/355212562c9eb26da4d2267ad13817e29857303d/example.png)
