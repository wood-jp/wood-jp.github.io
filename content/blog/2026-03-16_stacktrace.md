---
title: "Extended Error Use Case: stacktrace"
date: 2026-03-16
description: "Introducing stacktrace: attach a stacktrace to an error."
tags: ["golang", "errors", "generics", "logging"]
series: ["Go Error Handling"]
series_order: 4
draft: false
---

## Recap

At the start of this series, we created a generic error wrapper that allowed us to attach type-safe data to another error.

The important parts we will need to reference are:

```go
package xerrors

type ExtendedError[T any] struct {
    err  error
    Data T
}

func Extend[T any](data T, err error) error { ... }
func Extract[T any](err error) (T, bool) { ... }
```

One of the stated reasons for doing this was to allow structured logging of errors to easily contain additional structured data. In this article, we will further explore that particular use case with a new sub-package.

## Use Case: Where did this error come from?

Let's just admit it: stack traces have been around for a long, long time. It's much easier to debug an error when you can see the exact path the code took to reach said error.

### Problem Description

The most popular existing stack trace library is [pkg/errors](https://github.com/pkg/errors) which was archived on Dec 1, 2021. Dave Cheney is the legendary original author, and you can read more about why it was abandoned in this [GitHub issue](https://github.com/pkg/errors/issues/245).

{{< alert "comment" >}}
You might notice that Dave said *"I no longer use this package, in fact I no longer wrap errors."* Clearly I disagree, but to each their own.
{{< /alert >}}

### Ideal Solution

We have everything we need already. A stack trace is just data that we want to attach to an error, and `ExtendedError` does exactly that. For developers, it should be as easy as:

```go
return stacktrace.Wrap(err)
```

And if we do our jobs right, the stack trace will just magically appear when logging this returned error.

## Introducing The `stacktrace` Package

### What is a stack trace anyway?

Okay, so you might remember from your computer science classes this thing called *the stack* (as opposed to *the heap*), which you shouldn't confuse with the data structure of the same name. It is the same name because it uses that same data structure, but what it stores is information about where the program is running. This is called a `Frame`. A frame contains data that represents a single function call on the call stack. At the machine level, each frame is identified by a program counter which is a pointer into the machine code of that function.

As humans, we don't care more about the source code than machine code and pointers, so for our use case we will define a `Frame` as:

```go
type Frame struct {
    // File is the source file path of the frame.
    File string `json:"source"`
    // LineNumber is the line number within File where the call was made.
    LineNumber int `json:"line"`
    // Function is the fully-qualified function name of the frame.
    Function string `json:"func"`
}
```

And a `StackTrace` is then just:

```go
type StackTrace []Frame
```

Where the first frame is the most recent (the top of the stack).

### How to get a stacktrace at runtime?

The answer is in the question. There is a `runtime` package that provides some helpful funcs [Callers](https://pkg.go.dev/runtime#Callers) and [CallersFrames](https://pkg.go.dev/runtime#CallersFrames). This is our in, but you'll need to read some docs to know how to use it. If you do, you might come up with something like:

```go
func GetStack(skipFrames int) StackTrace {
    var stackTrace StackTrace

    pc := make([]uintptr, maxFrames)
    n := runtime.Callers(skipFrames, pc)
    pc = pc[:n]

    frames := runtime.CallersFrames(pc)
    for {
        frame, more := frames.Next()
        if !more {
            break
        }
        stackTrace = append(stackTrace, Frame{
            File:       frame.File,
            LineNumber: frame.Line,
            Function:   frame.Function,
        })
    }

    return stackTrace
}
```

The argument `skipFrames` will make sense in a few minutes. For now, pretend we always pass a zero.

{{< alert "lightbulb" >}}
It's worth comparing this to how the OG did it in [pkg/errors](https://github.com/pkg/errors/blob/master/stack.go). Keep in mind that `CallersFrames` didn't even exist at that time.
{{< /alert >}}

### Wrap it up

Now that we have a data type, and a way to fill it in, the rest is easy:

```go
const wrapStackDepth = 0
func Wrap(err error) error {
    if err == nil {
        return nil
    }
    // Only add a stacktrace if we don't already have one
    if _, ok := xerrors.Extract[StackTrace](err); !ok {
        return xerrors.Extend(GetStack(wrapStackDepth), err)
    }
    return err
}
```

and

```go
func Extract(err error) StackTrace {
    st, ok := xerrors.Extract[StackTrace](err)
    if !ok {
        return nil
    }
    return st
}
```

#### Try it out

{{< code-template id="stacktrace" >}}
package main

import (
    "errors"
    "fmt"
    "log/slog"
    "os"
    "runtime"
)

// xerrors (inlined)

type ExtendedError[T any] struct {
    err  error
    Data T
}

func (e ExtendedError[T]) Error() string { return e.err.Error() }
func (e ExtendedError[T]) Unwrap() error { return e.err }
func (e ExtendedError[T]) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Any("error", e.err),
        slog.Any("data", e.Data),
    )
}

func Extend[T any](data T, err error) error {
    if err == nil {
        return nil
    }
    return ExtendedError[T]{Data: data, err: err}
}

func Extract[T any](err error) (T, bool) {
    var e ExtendedError[T]
    ok := errors.As(err, &e)
    return e.Data, ok
}

// stacktrace (inlined)

const maxFrames = 32

type Frame struct {
    File       string `json:"source"`
    LineNumber int    `json:"line"`
    Function   string `json:"func"`
}

type StackTrace []Frame

func GetStack(skipFrames int) StackTrace {
    var st StackTrace
    pc := make([]uintptr, maxFrames)
    n := runtime.Callers(skipFrames, pc)
    pc = pc[:n]
    frames := runtime.CallersFrames(pc)
    for {
        frame, more := frames.Next()
        if !more {
            break
        }
        st = append(st, Frame{
            File:       frame.File,
            LineNumber: frame.Line,
            Function:   frame.Function,
        })
    }
    return st
}

const wrapStackDepth = 0

func Wrap(err error) error {
    if err == nil {
        return nil
    }
    if _, ok := Extract[StackTrace](err); !ok {
        return Extend(GetStack(wrapStackDepth), err)
    }
    return err
}

##CODE##
{{< /code-template >}}

{{< runnable sandbox="go" template="stacktrace" >}}
var ErrSentinel = fmt.Errorf("something went wrong")

func main() {
    err := Wrap(ErrSentinel)

    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    logger.Error("caught an error", slog.Any("error", err))
}
{{< /runnable >}}

If you run this, and prettify the output, you might notice there are 5 frames:

```json
{
    "source": "/usr/local/go/src/runtime/extern.go",
    "line": 345,
    "func": "runtime.Callers"
}
{
    "source": "/tmp/sandbox/main.go",
    "line": 55,
    "func": "main.GetStack"
}
{
    "source": "/tmp/sandbox/main.go",
    "line": 79,
    "func": "main.Wrap"
}
{
    "source": "/tmp/sandbox/main.go",
    "line": 87,
    "func": "main.main"
}
{
    "source": "/usr/local/go/src/runtime/proc.go",
    "line": 285,
    "func": "runtime.main"
}
```

#### skipFrames

Given that our call to `Wrap` and everything it does is not useful to a user, we don't want to see those. So that `skipFrames` I told you about earlier should be set to 3 not 0, in order to always ignore the frames for `runtime.Callers`, `GetStack`, and `Wrap` itself.

There's also that last frame, coming from `runtime.main` in this case, that we don't really need. From a developer point-of-view, the code starts with `main.main`. If you start writing tests, you'll also notice that `testing` funcs will clog up the stack too. To eliminate those, we can add a `skipRuntime` option:

```go
if skipRuntime {
    if strings.HasPrefix(frame.Function, runtimePrefix) || strings.HasPrefix(frame.Function, testingPrefix) {
        continue
    }
}
```

Where

```go
const (
    runtimePrefix = "runtime."
    testingPrefix = "testing."
)
```

Putting it all together, here is the complete `GetStack`:

```go
func GetStack(skipFrames int, skipRuntime bool) StackTrace {
    pc := make([]uintptr, maxFrames)
    n := runtime.Callers(skipFrames, pc)
    pc = pc[:n]

    stackTrace := make(StackTrace, 0, n)
    frames := runtime.CallersFrames(pc)
    for {
        frame, more := frames.Next()
        if !more {
            break
        }
        if skipRuntime {
            if strings.HasPrefix(frame.Function, runtimePrefix) || strings.HasPrefix(frame.Function, testingPrefix) {
                continue
            }
        }
        stackTrace = append(stackTrace, Frame{
            File:       frame.File,
            LineNumber: frame.Line,
            Function:   frame.Function,
        })
    }

    return stackTrace
}
```

And the updated `Wrap`, passing `skipFrames=3` to skip `runtime.Callers`, `GetStack`, and `Wrap` itself, as well as `skipRuntime=true` to drop the noise:

```go
const wrapStackDepth = 3

// Disabled disables stacktrace collection in Wrap when set to true.
var Disabled atomic.Bool

func Wrap(err error) error {
    if Disabled.Load() || err == nil {
        return err
    }
    if _, ok := xerrors.Extract[StackTrace](err); !ok {
        return xerrors.Extend(GetStack(wrapStackDepth, true), err)
    }
    return err
}
```

The `Disabled` flag is handy in tests where you want to suppress stack collection without conditionally calling `Wrap`.

### Logging the Stack Trace

We also need to implement `slog.LogValuer` for `StackTrace`:

```go
func (st StackTrace) LogValue() slog.Value {
    frames := make([]any, len(st))
    for i, frame := range st {
        frames[i] = map[string]any{
            "func":   frame.Function,
            "line":   frame.LineNumber,
            "source": frame.File,
        }
    }
    return slog.AnyValue(frames)
}
```

The end result looks something like:

```json
{
  "error": {
    "error": "something went wrong",
    "data": [
      {"func": "main.c", "line": 14, "source": ".../main.go"},
      {"func": "main.b", "line": 18, "source": ".../main.go"},
      {"func": "main.a", "line": 22, "source": ".../main.go"},
      {"func": "main.main", "line": 28, "source": ".../main.go"}
    ]
  }
}
```

{{< alert >}}
Using `map[string]any` and `slog.AnyValue` feels wrong here since we know the exact types of all the bits of data. However, `slog` handlers only resolve `slog.LogValuer` at the top level of an attribute value. They do not recursively resolve `slog.Value` elements. If we had used

```go
frames[i] = slog.GroupValue(
    slog.String("func", frame.Function),
    slog.Int("line", frame.LineNumber),
    slog.String("source", frame.File),
)
```

Then each of the items in the frame would unfortunately be rendered as empty objects (`{}`).
{{< /alert >}}

### Gotcha: Joined Errors

This is the same issue that we have talked about in every article in this series. In this case, it is quite easy to just think about what it would mean to join two or more errors that each have a stack trace. The `Extract` func can only return one, so which one? If you want to return them all, that means walking the tree (yes *tree* not just *chain*) and finding every single one.

It's not impossible, but if a developer really wants to join errors and also get stack traces, then let them figure out how to unjoin the errors. Dealing with the complexity of an error tree is just not worth the effort to me.

## Next Steps

The complete code for our stacktrace package can be found on [github](https://github.com/wood-jp/xerrors/tree/main/stacktrace). Note that the code will likely differ from what was presented here as it is expanded and improved over time.

### Related Reading

- [The Go runtime package](https://pkg.go.dev/runtime)
- The original: [pkg/errors](https://github.com/pkg/errors)
- [palantir/stacktrace](https://github.com/palantir/stacktrace)
- [quantumcycle/metaerr](https://github.com/quantumcycle/metaerr)
- [Getting stack traces for errors in Go](https://www.dolthub.com/blog/2023-11-10-stack-traces-in-go/)

---

_Some code in this article is based on work from [zkr-go-common](https://github.com/zircuit-labs/zkr-go-common-public), licensed under the MIT License, originally authored by [wood-jp](https://github.com/wood-jp) at [Zircuit](https://www.zircuit.com/)_
