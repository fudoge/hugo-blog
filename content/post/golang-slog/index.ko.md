---
title: "Go slog로 구조화된 로그 간단하게 남기기"
description: "Go 표준 라이브러리 log/slog의 기본 사용법부터 Handler, With, LogAttrs, Context, Group 활용법까지 예제로 알아본다"
date: 2026-09-20T00:45:23+09:00
lastmod: 2026-09-20T00:45:23+09:00
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

`log/slog`는 **Go 1.21부터 표준 라이브러리에 포함된 구조화 로깅 패키지**이다. 메시지와 함께 key-value 형태의 속성을 남길 수 있으며, API도 단순하고 간결하다.

표준 라이브러리이므로 새 프로젝트에서 별도의 서드 파티 로깅 패키지를 먼저 검토하지 않고도 시작하기 좋다. 호출부는 `slog` API로 유지하면서 내부 `Handler`만 교체할 수 있어, 나중에 다른 로깅 백엔드와 연동하기도 쉽다.

---

## 📝 slog 기본

`slog`는 로그 메시지와 속성을 **key-value 형식**으로 기록한다.

기본 형태는 `slog.LEVEL(message, key1, value1, key2, value2)`이며, `time`, `level`, `msg`는 자동으로 출력된다.

아래 예시는 기본 logger를 사용한다.

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

## ⚙️ Handler 지정하기

규모가 있는 애플리케이션에서는 기본 logger를 바로 사용하기보다 **`Handler`를 지정한 logger를 생성하고 의존성으로 전달하는 방식**이 유용하다. 전역 logger에만 의존하면 코드에서 로깅 의존성을 파악하기 어렵고, 컴포넌트마다 필요한 설정이나 속성을 적용하기도 까다로워지기 때문이다.

```go
logger := slog.New(
	slog.NewTextHandler(os.Stdout, nil),
)

logger.Info("server started",
	"addr", ":8080",
)
```

### TextHandler

사람이 읽기 쉬운 단순 텍스트 형태로 로그를 출력한다.

```go
logger := slog.New(
	slog.NewTextHandler(os.Stdout, nil),
)
```

출력 결과는 다음과 같다.

```bash
time=2026-09-17T01:00:00.000+09:00 level=INFO msg="server started" addr=:8080
```

### JSONHandler

로그 수집기에서 파싱하기 쉬운 JSON 형식으로 출력한다.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, nil),
)
```

출력 결과는 다음과 같다.

```json
{"time":"2026-09-17T01:00:00+09:00","level":"INFO","msg":"server started","addr":":8080"}
```

### HandlerOptions

Handler를 생성할 때 `slog.HandlerOptions`로 로그 레벨, 소스 위치, 속성 치환 규칙 등을 설정할 수 있다. 아래 예시는 **`AddSource: true`로 로그를 호출한 파일과 줄 번호를 함께 출력**한다.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		AddSource: true,
	}),
)
```

### SetDefault

전역 logger가 항상 나쁜 것은 아니다. 작은 서버나 CLI 도구처럼 로깅 정책이 하나로 통일된 프로그램이라면 기본 logger가 더 간단할 수 있다. `slog.SetDefault`를 사용하면 패키지 수준의 로깅 함수가 사용할 logger를 교체할 수 있다.

```go
logger := slog.New(
	slog.NewJSONHandler(os.Stdout, nil),
)

slog.SetDefault(logger)
```

이후에는 패키지 수준의 함수로 같은 설정을 사용할 수 있다.

```go
slog.Info("hello")
slog.Info("failed", "err", err)
```

---

## 🔁 With로 반복 줄이기

`With`는 여러 로그에 공통으로 들어가는 속성을 미리 묶어 새로운 logger를 만든다. 아래처럼 `request_id`와 `user_id`를 매번 작성하면 코드가 길어지고, 일부 로그에서 속성을 빠뜨리기 쉽다.

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

대신 공통 속성을 `With`로 한 번만 지정할 수 있다. 새로 만든 `log`는 기존 `logger`의 설정을 유지하면서 **`request_id`와 `user_id`를 모든 로그에 자동으로 포함**한다.

```go
log := logger.With(
	"request_id", requestID,
	"user_id", userID,
)

// request_id와 user_id는 이미 포함되어 있다.
log.Info("request started")
log.Info("query executed")
log.Info("request finished")
```

이 방식은 요청 단위의 정보가 많은 서버에서 특히 유용하다. 요청을 받을 때 logger에 공통 속성을 추가한 뒤, 해당 요청을 처리하는 하위 계층으로 전달할 수 있다.

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

## 🧩 Attribute 타입 명시하기

가장 간단한 방법은 key와 value를 번갈아 전달하는 것이다.

```go
logger.Info("user logged in",
	"user_id", 42,
	"admin", true,
)
```

`slog.Int`, `slog.Bool`, `slog.String`과 같은 생성자를 사용하면 속성의 타입을 명시할 수 있다.

```go
logger.Info("user logged in",
	slog.Int("user_id", 42),
	slog.Bool("admin", true),
	slog.String("ip", "127.0.0.1"),
)
```

속성이 많거나 할당 비용을 줄이고 싶다면 `LogAttrs`를 사용할 수 있다. `...any` 대신 `slog.Attr`를 직접 전달하므로 **타입이 명확하고 불필요한 변환 비용을 줄일 수 있다.**

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

## 🧵 Context 포함하기

`InfoContext`, `WarnContext`, `ErrorContext`, `LogAttrs`를 사용하면 로그 호출에 `context.Context`를 전달할 수 있다. 커스텀 Handler는 이 context에서 trace ID나 span ID 같은 요청 정보를 꺼내 속성으로 추가할 수 있다.

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

다만 **context가 취소되었다고 해서 로그 기록 자체가 자동으로 중단되지는 않는다.** `context.Context`는 Handler가 요청 범위의 정보를 참조할 수 있도록 전달되는 값이며, 로그 출력 여부는 Handler의 구현과 설정된 로그 레벨이 결정한다.

---

## 🗂️ Group으로 속성 묶기

`slog.Group`을 사용하면 관련된 속성을 하나의 그룹으로 묶을 수 있다. 요청과 응답처럼 이름이 충돌하기 쉬운 속성을 **명확한 계층 구조로 표현**할 때 유용하다.

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

JSON Handler에서는 다음과 같이 중첩된 객체로 출력된다.

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

Text Handler에서는 같은 구조가 `request.method=GET`과 같은 dot notation으로 출력된다.

---

## 🌐 HTTP 서버 적용 예시

다음은 앞에서 살펴본 `With`, `Group`, 타입이 지정된 Attribute를 HTTP webhook handler에 적용한 예시이다.

요청 정보는 처음에 한 번 묶고, 처리 결과에 따라 **응답 상태와 오류를 같은 구조로 기록**한다. 이렇게 하면 로그 수집 시스템에서 `request.method`, `response.status` 같은 필드로 검색하거나 집계하기 쉽다.

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

이 예시에서 로그 레벨은 다음 기준으로 나누었다.

- 정상 처리와 인증 실패처럼 운영 중 예상할 수 있는 결과는 `Info`
- 잘못된 payload나 요청 크기 초과처럼 확인이 필요한 입력 문제는 `Warn`
- 외부 서비스 장애나 예상하지 못한 내부 오류는 `Error`

핵심은 모든 로그에 많은 정보를 무조건 넣는 것이 아니라, **검색과 문제 해결에 필요한 필드를 일관된 이름과 구조로 남기는 것**이다.

---

## 📚 References

- [Structured Logging with `slog`](https://go.dev/blog/slog)
- [`log/slog` package documentation](https://pkg.go.dev/log/slog)
- [`slog` Handler Guide](https://pkg.go.dev/golang.org/x/example/slog-handler-guide)
- [`fudoge/ntfy-gateway`](https://github.com/fudoge/ntfy-gateway)
