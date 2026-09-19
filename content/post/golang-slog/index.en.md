---
title: "Structured Logging in Go with slog"
description: "Learn how to use Go's log/slog package, from handlers and contextual attributes to LogAttrs, groups, and a practical HTTP server example"
date: 2026-09-20T00:45:26+09:00
lastmod: 2026-09-20T00:45:26+09:00
slug: golang-slog
image:
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - Go
    - Golang
    - slog
    - Logging
    - Observability

categories:
    - Golang
---

`log/slog` is a **structured logging package included in the Go standard library since Go 1.21**. It records key-value attributes alongside a message while keeping the API simple and concise.

Because it is part of the standard library, `slog` is a sensible starting point for a new project without immediately reaching for a third-party logging package. Application code can continue to use the `slog` API while the underlying `Handler` is replaced later to integrate with another logging backend.

---

## 📝 slog basics

`slog` records a log message and its attributes in **key-value form**.

The basic pattern is `slog.LEVEL(message, key1, value1, key2, value2)`. The `time`, `level`, and `msg` fields are included automatically.

The following example uses the default logger.

```go
package main

import (
	"log/slog"
)

func main() {
	slog.Info("server started",
		"addr", ":8080",
		"env", "production",
	)

	slog.Warn("request slow",
		"elapsed_ms", 1200,
	)

	slog.Error("database failed",
		"err", "connection refused",
	)
}
```

---

## ⚙️ Configuring a Handler

For a larger application, it is often useful to **create a logger with an explicit `Handler` and pass it as a dependency** instead of using the default logger everywhere. Relying only on a global logger makes the logging dependency less visible and makes component-specific configuration or attributes harder to apply.

```go
logger := slog.New(
	slog.NewTextHandler(os.Stdout, nil),
)

logger.Info("server started",
	"addr", ":8080",
)
```

### TextHandler

`TextHandler` writes logs in a simple format that is easy for people to read.

```go
logger := slog.New(
	slog.NewTextHandler(os.Stdout, nil),
)
```

The output looks like this:

```bash
time=2026-09-17T01:00:00.000+09:00 level=INFO msg="server started" addr=:8080
```

### JSONHandler

`JSONHandler` writes logs as JSON, which is easier for log collectors to parse.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, nil),
)
```

The output looks like this:

```json
{"time":"2026-09-17T01:00:00+09:00","level":"INFO","msg":"server started","addr":":8080"}
```

### HandlerOptions

When creating a Handler, `slog.HandlerOptions` can configure the minimum log level, source location, and attribute replacement rules. In the following example, **`AddSource: true` includes the file and line number of the log call**.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		AddSource: true,
	}),
)
```

### SetDefault

A global logger is not always a poor choice. It can be convenient for a small server or CLI tool with a single logging policy. `slog.SetDefault` replaces the logger used by package-level logging functions.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, nil),
)

slog.SetDefault(logger)
```

The package-level functions will then use the same configuration.

```go
slog.Info("hello")
slog.Info("failed", "err", err)
```

---

## 🔁 Reducing repetition with With

`With` creates a new logger with a set of attributes that should appear in multiple log records. Repeating `request_id` and `user_id` in every call makes the code noisy and makes it easy to omit an attribute from one of the records.

```go
logger.Info("request started",
	"request_id", requestID,
	"user_id", userID,
)

logger.Info("query executed",
	"request_id", requestID,
	"user_id", userID,
)
```

Instead, define the shared attributes once with `With`. The new `log` value preserves the configuration of `logger` while **automatically including `request_id` and `user_id` in every record**.

```go
log := logger.With(
	"request_id", requestID,
	"user_id", userID,
)

// request_id and user_id are already included.
log.Info("request started")
log.Info("query executed")
log.Info("request finished")
```

This pattern is especially useful in servers, where many attributes belong to a specific request. Add those attributes when the request arrives, then pass the derived logger to the layers that process it.

```go
func handler(logger *slog.Logger) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		log := logger.With(
			"method", r.Method,
			"path", r.URL.Path,
			"remote_addr", r.RemoteAddr,
		)

		log.Info("request started")

		// ...

		log.Info("request completed")
	}
}
```

---

## 🧩 Using typed attributes

The simplest form alternates keys and values.

```go
logger.Info("user logged in",
	"user_id", 42,
	"admin", true,
)
```

Constructors such as `slog.Int`, `slog.Bool`, and `slog.String` make each attribute's type explicit.

```go
logger.Info("user logged in",
	slog.Int("user_id", 42),
	slog.Bool("admin", true),
	slog.String("ip", "127.0.0.1"),
)
```

When a record has many attributes or allocations matter, use `LogAttrs`. It accepts `slog.Attr` values instead of `...any`, providing **explicit types while avoiding unnecessary conversions**.

```go
logger.LogAttrs(
	ctx,
	slog.LevelInfo,
	"request completed",
	slog.String("method", r.Method),
	slog.String("path", r.URL.Path),
	slog.Int("status", 200),
)
```

---

## 🧵 Passing a Context

`InfoContext`, `WarnContext`, `ErrorContext`, and `LogAttrs` accept a `context.Context`. A custom Handler can use it to retrieve request-scoped information such as a trace ID or span ID and add that information to a record.

```go
func handleRequest(ctx context.Context, logger *slog.Logger, requestID string) {
	logger.InfoContext(ctx, "request started",
		slog.String("request_id", requestID),
	)

	// ...

	logger.InfoContext(ctx, "request completed",
		slog.String("request_id", requestID),
	)
}
```

However, **canceling the context does not automatically cancel the log operation**. The context gives the Handler access to request-scoped data; the Handler implementation and configured log level determine whether the record is emitted.

---

## 🗂️ Grouping attributes

`slog.Group` collects related attributes under one name. It is useful for representing data such as requests and responses in a **clear hierarchy without field-name collisions**.

```go
logger.Info("request completed",
	slog.Group("request",
		"method", r.Method,
		"path", r.URL.Path,
	),
	slog.Group("response",
		"status", 200,
		"bytes", 1024,
	),
)
```

A JSON Handler writes the groups as nested objects.

```json
{
  "level": "INFO",
  "msg": "request completed",
  "request": {
    "method": "GET",
    "path": "/users"
  },
  "response": {
    "status": 200,
    "bytes": 1024
  }
}
```

A Text Handler represents the same structure with dot notation, such as `request.method=GET`.

---

## 🌐 HTTP server example

The following example applies `With`, `Group`, and typed attributes to an HTTP webhook handler.

Request attributes are grouped once at the beginning, while each outcome records the **response status and error using a consistent structure**. This makes it easy to search or aggregate records by fields such as `request.method` and `response.status` in a log collection system.

```go
// source: https://github.com/fudoge/ntfy-gateway
func handleWebhook(gateway Dispatcher, logger *slog.Logger, w http.ResponseWriter, r *http.Request) {
	sourceID := r.PathValue("source")
	logger = logger.With(
		slog.Group(
			"request",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
		),
	)

	if !isJSON(r.Header.Get("Content-Type")) {
		msg := "content type must be application/json"
		logger.Info(
			msg,
			slog.Group("response", slog.Int("status", http.StatusUnsupportedMediaType)),
		)
		http.Error(w, msg, http.StatusUnsupportedMediaType)
		return
	}

	credential, ok := bearerToken(r.Header.Get("Authorization"))
	if !ok {
		w.Header().Set("WWW-Authenticate", "Bearer")
		msg := "unauthorized"
		logger.Info(
			msg,
			slog.Group("response", slog.Int("status", http.StatusUnauthorized)),
		)
		http.Error(w, msg, http.StatusUnauthorized)
		return
	}

	r.Body = http.MaxBytesReader(w, r.Body, maxWebhookBodySize)

	err := gateway.Dispatch(r.Context(), sourceID, credential, r.Body)
	if err == nil {
		logger.Info(
			"webhook dispatched",
			slog.Group("response", slog.Int("status", http.StatusAccepted)),
		)
		w.WriteHeader(http.StatusAccepted)
		return
	}

	var maxBytesError *http.MaxBytesError
	switch {
	case errors.Is(err, service.ErrUnauthorized):
		w.Header().Set("WWW-Authenticate", "Bearer")
		msg := "unauthorized"
		logger.Info(
			msg,
			slog.Group("response", slog.Int("status", http.StatusUnauthorized)),
			slog.Any("error", err),
		)
		http.Error(w, msg, http.StatusUnauthorized)
	case errors.As(err, &maxBytesError):
		msg := "request body too large"
		logger.Warn(
			msg,
			slog.Group("response", slog.Int("status", http.StatusRequestEntityTooLarge)),
			slog.Any("error", err),
		)
		http.Error(w, msg, http.StatusRequestEntityTooLarge)
	case errors.Is(err, service.ErrInvalidPayload):
		msg := "invalid payload"
		logger.Warn(
			msg,
			slog.Group("response", slog.Int("status", http.StatusBadRequest)),
			slog.Any("error", err),
		)
		http.Error(w, msg, http.StatusBadRequest)
	case errors.Is(err, service.ErrPublish):
		logger.Error(
			"failed to publish webhook",
			slog.Group("response", slog.Int("status", http.StatusBadGateway)),
			slog.Any("error", err),
		)
		http.Error(w, "upstream notification service failed", http.StatusBadGateway)
	default:
		logger.Error(
			"failed to dispatch webhook",
			slog.Group("response", slog.Int("status", http.StatusInternalServerError)),
			slog.Any("error", err),
		)
		http.Error(w, "internal server error", http.StatusInternalServerError)
	}
}
```

The example uses the following log-level policy:

- `Info` for successful operations and expected outcomes such as authentication failures
- `Warn` for input problems that deserve attention, such as an invalid payload or oversized request body
- `Error` for an upstream service failure or an unexpected internal error

The goal is not to attach as much data as possible to every record. It is to **record the fields needed for searching and troubleshooting with consistent names and structure**.

---

## 📚 References

- [Structured Logging with `slog`](https://go.dev/blog/slog)
- [`log/slog` package documentation](https://pkg.go.dev/log/slog)
- [`slog` Handler Guide](https://pkg.go.dev/golang.org/x/example/slog-handler-guide)
- [`fudoge/ntfy-gateway`](https://github.com/fudoge/ntfy-gateway)
