---
title: "Extended Error Use Case: errcontext"
date: 2026-02-25
description: "Introducing errcontext: attach structured logging to an error chain."
tags: ["golang", "errors", "generics", "logging"]
series: ["Go Error Handling"]
series_order: 3
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

## Use Case: Error Logging Context

Let's attach data specifically meant to enhance the logging of an error.

### Problem Description

You might recall that we already have a nice way to log additional data with our errors since we implemented `LogValue`:

```go
func (e ExtendedError[T]) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Any("error", e.err),
        slog.Any("data", e.Data),
    )
}
```

However, this requires code authors to create a data type `T` to pass into wrapping the error. It also forces each frame in the call stack to do its own wrapping, possibly resulting in redundant or difficult to obtain data. For example, if `foo()` calls `bar()` and both wrap a resulting error with the same type `Baz`, then ultimately both instances of `Baz` will be attached and logged. This might or might not be the desired result, but there is no choice in the matter from the perspective of the code author.

### Ideal Solution

As a developer, it would be nice to be able to write something like:

```go
func foo(a, b, c int) error {
    if err := bar(c, "hello log"); err != nil {
        return errcontext.Add(err, slog.Int("a", a), slog.Int("b", b), slog.Int("c", c))
    }
}

func bar(c int, d string) error {
    return errcontext.Add(err, slog.Int("c", c), slog.String("d", d))
}
```

Then when logging the error from `foo`, expect that log to contain `a`, `b`, and `c` values all together, and just once each.

This leverages an existing type, [slog.Attr](https://pkg.go.dev/log/slog#Attr), and provides a simple and flexible API that is similar to the logging we actually want to do.

## Introducing The `errcontext` Package

An `slog.Attr` is just

```go
type Attr struct {
    Key   string
    Value Value
}
```

Which means that in order to collect multiple such values, we can simply use a map as our data type.

```go
type Context map[string]slog.Value
```

And our API should be simple as well:

```go
func Add(err error, context ...slog.Attr) error
func Get(err error) Context
func (c Context) LogValue() slog.Value
func (c Context) Flatten() []slog.Attr
```

One might also consider a `Delete` func, but I'd say YAGNI (you aren't gonna need it): if there is something that shouldn't be logged, don't `Add` it in the first place.

The `Flatten` func isn't 100% required, but I'll talk about why I've included it when we get to its implementation [below](#logvalue-and-flatten).

### Get

The `Get` func is just a convenience wrapper around `xerrors.Extract[Context]`:

```go
func Get(err error) Context {
    if err == nil {
        return nil
    }

    if context, ok := xerrors.Extract[Context](err); ok {
        return context
    }
    return nil
}
```

### Add

The `Add` func at its core is also simple:

```go
func Add(err error, context ...slog.Attr) error {
    // ...
    newContext := make(Context, len(context))
    for _, attr := range context {
        newContext[attr.Key] = attr.Value
    }
    return xerrors.Extend(newContext, err)
}
```

However we should handle the basic edge cases:

```go
    if err == nil {
        return nil
    } else if len(context) == 0 {
        return err
    }
```

There's also another important case to consider: what if the error has already been wrapped with a `Context`? This is exactly the situation we have in our example in [Ideal Solution](#ideal-solution). If we leave the `Add` func as-is, this would cause multiple instances of `Context` to be wrapped with the error, and our `Get` would only return the closest one in the chain.

Since the `Context` type is a map, the data we are adding to the wrapped error is just a pointer to the map data, and we can actually modify it directly:

```go
    if existing := Get(err); existing != nil {
        for _, attr := range context {
            existing[attr.Key] = attr.Value
        }
        return err
    }
```

Now there will only be a single `Context` in the error chain, attached to the first error that was wrapped.

{{< alert >}}
Note that this will also cause duplicate `attr.Key` values to be overwritten up the chain, and the original values cannot be recovered. This is an intentional design decision that keeps `errcontext` simple. If the original data is different and important, wrap that data in a different way.
{{< /alert >}}

### LogValue and Flatten

The `LogValue` func can be made extremely simple by leveraging `Flatten`:

```go
func (c Context) LogValue() slog.Value {
    if len(c) == 0 {
        return slog.Value{}
    }
    return slog.GroupValue(c.Flatten()...)
}
```

If everything is being done correctly, there should never be an empty `Context`, but there's nothing stopping someone from using our `Context` differently than intended.

Let's talk about `Flatten`. `Flatten` will actually return a `[]slog.Attr` that is sorted by key. This func will serve two important uses: consistency in logs; and easier testing. In the case of logging, having logged values always appear in the same order is the kind of nicety you don't realize you actually need until you start scrolling through logs with non-deterministic ordering and start losing your mind. Trust me, you don't want that. The bonus of this as a separate func means we can also write tests for deterministic outputs; we don't need to sort the outputs ourselves in the tests for comparison.

But wait, you might say, isn't sorting going to slow this down? No. No sane person is going to attach so many log attributes that sorting will make any kind of actual impact on performance. And _in my opinion_, if performance is that important, logging is probably a bad idea in the first place. If this topic interests you, start by reading more about [slog's performance](https://www.dash0.com/guides/logging-in-go-with-slog#the-performance-question-is-slog-good-enough).

### Gotcha: Joined Errors

This is the same issue that we talked about in the [previous article]({{< ref "/blog/2026-02-20_errclass" >}}#gotcha-joined-errors). Using this package on a joined error is just asking for trouble, and implementing a solution is more trouble than I want to deal with at this time. So, once again, I instead accept that joined errors are not supported and just leave it at that.

## Next Steps

The complete code for our errcontext package can be found on [github](https://github.com/wood-jp/xerrors/tree/main/errcontext). Note that the code will likely differ from what was presented here as it is expanded and improved over time.

### Related Reading

- [slog.Attr Docs](https://pkg.go.dev/log/slog#Attr)
- [Logging in Go with Slog: A Practitioner's Guide - Performance](https://www.dash0.com/guides/logging-in-go-with-slog#the-performance-question-is-slog-good-enough)

---

_Some code in this article is based on work from [zkr-go-common](https://github.com/zircuit-labs/zkr-go-common-public), licensed under the MIT License, originally authored by [wood-jp](https://github.com/wood-jp) at [Zircuit](https://www.zircuit.com/)_
