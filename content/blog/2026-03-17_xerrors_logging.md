---
title: "Extended Error Logging"
date: 2026-03-17
description: "Improving the logging of Extended Errors"
tags: ["golang", "errors", "generics", "logging"]
series: ["Go Error Handling"]
series_order: 5
draft: false
---

## Recap

We have created a high-level `ExtendedError` type, and we implemented [slog.LogValuer](https://pkg.go.dev/log/slog#LogValuer) with the following code:

```go
func (e ExtendedError[T]) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Any("error", e.err),
        slog.Any("data", e.Data),
    )
}
```

Each of our sub-packages (`errclass`, `errcontext`, and `stacktrace`) also implement `slog.LogValuer`. This allows us to add details to our logs easily.

## Problem Description

Consider the following program:

```go
func c() error {
    return stacktrace.Wrap(errclass.WrapAs(errors.New("something went wrong"), errclass.Transient))
}

func b() error {
    return c()
}

func a() error {
    return b()
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    err := a()
    logger.Error("request failed", slog.Any("error", err))
}
```

Which would output something like:

```json
{
  "level": "ERROR",
  "msg": "request failed",
  "error": {
    "error": {
      "error": "something went wrong",
      "data": {
        "class": "transient"
      }
    },
    "data": [
      {
        "func": "main.c",
        "line": 14,
        "source": ".../main.go"
      },
      {
        "func": "main.b",
        "line": 18,
        "source": ".../main.go"
      },
      {
        "func": "main.a",
        "line": 22,
        "source": ".../main.go"
      },
      {
        "func": "main.main",
        "line": 28,
        "source": ".../main.go"
      }
    ]
  }
}
```

If changed func `c` to wrap the stacktrace before the class, the ouput would be similar, but the nesting would be reversed. In general, this kind of nesting is not that helpful for logging.

### Ideal Solution

Really we want our log output to be more like this:

```json
{
  "level": "ERROR",
  "msg": "request failed",
  "error": {
    "error": "something went wrong",
    "error_detail": {
      "class": "transient",
      "stacktrace": [...]
    }
  }
}
```

Where each type of error appears at the same level with an identifying key. In fact, it might be even better if it was like this:

```json
{
  "level": "ERROR",
  "msg": "request failed",
  "error": "something went wrong",
  "error_detail": {
    "class": "transient",
    "stacktrace": [...]
  }
}
```

However, you'll recall that implementing `slog.LogValuer` requires the func `LogValue() slog.Value` so there's just no way to directly return two keys (`"error"` and `"error_detail"`) withouth grouping them as we have.

{{< alert "comment" >}}
It actually _is_ possible, but requires a custom `slog.Handler` and falls outside of the scope of the `xerrors` package. I had some debate on this topic in the past, where there was a concern that log aggregators might not like the error message being buried under `error.error`, but what I have found in practice is that log aggregators care more about the `msg` and `level` keys. Anything else requires some work to properly index regardless.
{{< /alert >}}

### Flattening our Logs

`LogValue` has each `ExtendedError` layer build its own nested value. Every wrapper adds another level of nesting, and the order depends entirely on which wrap happened first. Instead, we will take a new approach which separates two concerns:

1. Each layer declares its **flat** contribution via `flatLogAttrs()`
2. A central `collectDetails()` walks the entire chain and gathers everything at one level

To support this, we introduce an unexported interface:

```go
type extendedErrFlat interface {
    flatLogAttrs() []slog.Attr
    innerError() error
}
```

It is unexported because only `ExtendedError[T]` should implement it. Callers interact only through the `slog.LogValuer` interface on the error itself.

#### `flatLogAttrs()`

This is where the interesting work happens. If `T` implements `slog.LogValuer` and its resolved value is a `KindGroup`, we unwrap those attrs directly instead of nesting them under a `"data"` key:

```go
func (e ExtendedError[T]) flatLogAttrs() []slog.Attr {
    val := slog.AnyValue(e.Data)
    for val.Kind() == slog.KindLogValuer {
        val = val.LogValuer().LogValue()
    }
    if val.Kind() == slog.KindGroup {
        return val.Group()
    }
    return []slog.Attr{slog.Any("data", e.Data)}
}
```

This allows for some simple updates to each of our sub-packages:

##### errclass

Wrap the output in a `"class"` group:

```go
func (c Class) LogValue() slog.Value {
    return slog.GroupValue(
        slog.String("class", c.String()),
    )
}
```

##### errcontext

Wrap the output in a `"context"` group:

```go
func (c Context) LogValue() slog.Value {
    if len(c) == 0 {
        return slog.GroupValue()
    }
    return slog.GroupValue(slog.Attr{Key: "context", Value: slog.GroupValue(c.Flatten()...)})
}
```

##### stacktrace

Wrap the output in a `"stacktrace"` group:

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
    return slog.GroupValue(slog.Any("stacktrace", slog.AnyValue(frames)))
}
```

#### `collectDetails()`

```go
func collectDetails(err error) []slog.Attr {
    if err == nil {
        return nil
    }
    if ee, ok := err.(extendedErrFlat); ok {
        inner := collectDetails(ee.innerError())
        return append(inner, ee.flatLogAttrs()...)
    }
    if u := errors.Unwrap(err); u != nil {
        return collectDetails(u)
    }
    return nil
}
```

This recursively walks the error chain, collecting attrs innermost-first. The outermost wrapper's attrs appear last, consistent with how error chains typically read. The second `errors.Unwrap` branch means plain `fmt.Errorf("%w", ...)` wrappers are transparent: `collectDetails` passes through them to find any `ExtendedError` layers below.

#### Updated `LogValue` and `logValue`

`ExtendedError[T].LogValue()` now delegates to a shared `logValue` func instead of doing its own nesting:

```go
func logValue(err error) slog.Value {
    detailAttrs := collectDetails(err)
    result := []slog.Attr{slog.String("error", err.Error())}
    if len(detailAttrs) > 0 {
        result = append(result, slog.Attr{
            Key:   "error_detail",
            Value: slog.GroupValue(detailAttrs...),
        })
    }
    return slog.GroupValue(result...)
}
```

All layers now contribute to one `error_detail` group rather than each building its own nested structure. The `error_detail` key only appears when there is something to report.

#### `xerrors.Log`

The package also provides a convenience function for the common case where the caller holds a plain `error` interface and does not have an `ExtendedError` in hand:

```go
func Log(err error) slog.Attr {
    return slog.Any("error", logValue(err))
}
```

Usage:

```go
logger.Error("request failed", xerrors.Log(err))
```

This produces the same flat output as calling `slog.Any("error", err)` on an `ExtendedError`, but works for any error in the chain.

#### The result

Going back to the original example:

```go
func c() error {
    return stacktrace.Wrap(errclass.WrapAs(errors.New("something went wrong"), errclass.Transient))
}
```

The log output is now:

```json
{
  "level": "ERROR",
  "msg": "request failed",
  "error": {
    "error": "something went wrong",
    "error_detail": {
      "class": "transient",
      "stacktrace": [
        {"func": "main.c", "line": 14, "source": ".../main.go"},
        {"func": "main.b", "line": 18, "source": ".../main.go"},
        {"func": "main.a", "line": 22, "source": ".../main.go"},
        {"func": "main.main", "line": 28, "source": ".../main.go"}
      ]
    }
  }
}
```

`class` and `stacktrace` sit as siblings at the same level under `error_detail`, regardless of which order the wraps happened in. Swap the order of `stacktrace.Wrap` and `errclass.WrapAs` in `c()` and the output is semantically identical (the order in which keys appear would still change).

## Next Steps

The complete code for the xerrors package can be found on [github](https://github.com/wood-jp/xerrors). Note that the code will likely differ from what was presented here as it is expanded and improved over time.

---

_Some code in this article is based on work from [zkr-go-common](https://github.com/zircuit-labs/zkr-go-common-public), licensed under the MIT License, originally authored by [wood-jp](https://github.com/wood-jp) at [Zircuit](https://www.zircuit.com/)_
