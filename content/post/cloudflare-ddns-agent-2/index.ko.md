---
title: "Cloudflare DDNS client 보완하기"
description: "이전에 만들었던 Cloudflare DDNS Agent를 보완한 DDNS 레코드 자동화 도구 개발"
date: 2026-08-03T20:21:41+09:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false

Categories:
- Homelab

Tags:
- Homelab
- Cloudflare
- Golang
---

## ℹ️ 개요

[이전]({{< relref "post/cloudflare-ddns-agent" >}})에 Cloudflare API와 Golang을 이용해서 간단한 프로그램을 만들어 DDNS 레코드 업데이트를 자동화하는 데 썼다. \
그러나, 이 때의 코드의 경우 옵션들도 제한적이었고, 코드 구조도 더 깔끔하게 정리하고 싶었다.

이번에 홈랩을 개편하기 시작한 김에 간단한 Cloudflare DDNS 레코드를 업데이트하는 Go 프로그램을 조금은 제대로 된 프로그램으로 개편해보고자 한다.


## 📁 폴더 구조

```text
.
├── cmd
│   └── cfddns
│       └── main.go
├── internal
│   ├── cloudflare
│   │   └── client.go
│   ├── config
│   │   ├── config.go
│   │   └── config_test.go
│   ├── ddns
│   │   ├── sync.go
│   │   └── sync_test.go
│   └── publicip
│       ├── client.go
│       └── client_test.go
├── Dockerfile
├── go.mod
├── go.sum
├── LICENSE
├── Makefile
└── README.md
```

기존에는 루트에 단일 `main.go`만 유지했다면, 이제는 제대로 된 Golang 프로젝트의 형식을 갖추었다. \
실행 프로그램의 진입점을 `cmd/`에 넣고, 내부 로직을 `interanl`에 두었다.

각각의 다음의 역할을 맡는다 :
- `internal/cloudflare`은 Cloudflare API wrapper
- `internal/config`은 flag 및 환경변수 값 불러오기
- `internal/ddns`에는 실제 모드별 로직 
- `internal/publicip`에는 Public IP를 로드

또한, 테스트 코드를 통해 코드를 점검한다.

## 🌐 유연한 Public IP Provider 제공

이전에는 단순히 Ipify에서만 가져올 수 있었지만, 이제는 여러 선택지가 주어진다. \
이제는 Ipify에서 문제가 있을 때, 다른 Provider를 이용할 수 있다. \
두 가지 유형의 Public IP를 이용할 수 있도록 개선했다. 

- PlainText응답: 응답이 단순히 IP주소인 경우
- JSON 응답: 응답이 JSON으로 반환되는 경우

CLI명령어에서 `--jsonpath "$.ip"`와 같이 플래그에 값이 주어지면, `JSONClient`를 통해 요청하고, 기본적으로 해당 플래그를 사용하지 않으면 `PlainClient`를 이용한다. \
`--endpoint`를 명시하지 않을 시, 기본적으로 Ipify에 요청하도록 하였다.

각각의 사용 예시는 다음과 같다:
```bash
# Plain text response, compatible with the default ipify endpoint.
cfddns --endpoint https://api.ipify.org

# JSON response.
cfddns --endpoint 'https://api64.ipify.org?format=json' --jsonpath '$.ip'
```

`PlainClient` struct와 `JSONClient` struct는 각각 `IPResolver` 인터페이스를 구현하도록 하였다.

```go
// internal/publicip/client.go
type IPResolver interface {
	Resolve(ctx context.Context, timeout time.Duration) (netip.Addr, error)
}

type PlainClient struct {
	endpoint string
}

type JSONClient struct {
	endpoint string
	jsonPath string
}

func (c PlainClient) Resolve(ctx context.Context, timeout time.Duration) (netip.Addr, error) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	// HTTP GET -> trim response body -> netip.ParseAddr
}

func (c JSONClient) Resolve(ctx context.Context, timeout time.Duration) (netip.Addr, error) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	// HTTP GET -> parse JSON -> extract JSONPath -> netip.ParseAddr
}

func GetIP(ctx context.Context, options *Options) (netip.Addr, error) {
	if options.ResponseType == config.ResponseTypePlain {
		return NewPlainClient(options.Endpoint).Resolve(ctx, options.Timeout)
	}

	return NewJSONClient(options.Endpoint, options.JSONPath).Resolve(ctx, options.Timeout)
}
```

## ⚙️ 모드 지원 추가

기존에는 단순히 현재의 레코드를 조회하고, 있으면 업데이트하고 없으면 새로 추가하는 방식이었다.

당장에는 나도 `replace`만 사용하지만, 혹시라도 같은 도메인 이름을 여러 public IP기반으로 홈랩을 운영하고, 실제로 트래픽을 DNS레벨에서 분산시키고자 하는 목적을 가진 사람이 있다면, `append`모드도 필요할 수 있을 거라고 생각했다.

이제는 더 세분화된 로직을 사용한다.

- `replace` 모드: 현재의 Public IP만 남기며, 기본으로 동작
- `append` 모드: 해당 도메인 이름이 여러 record를 가질 수 있도록 동작

또한, 이제는 TTL값도 기본만을 사용하지 않고 커스텀할 수 있어 기존 레코드가 있더라도 TTL이 다르면 Update하여 같은 IP값이더라도 TTL을 업데이트시키도록 한다.

```go
// internal/ddns/sync.go
type DNSClient interface {
	ListARecords(ctx context.Context, name string) ([]cloudflare.ARecord, error)
	CreateARecord(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error
	UpdateARecord(ctx context.Context, recordID, name string, ip netip.Addr, ttl dns.TTL) error
	DeleteARecord(ctx context.Context, recordID string) error
}

func (s *Syncer) Append(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error {
	// 같은 IP의 A record가 있으면 TTL만 보정한다.
	// 없으면 새 A record를 추가한다.
}

func (s *Syncer) Replace(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error {
	// 현재 Public IP와 다른 A record는 삭제한다.
	// 같은 IP의 record가 여러 개라면 하나만 남긴다.
	// 남긴 record의 TTL이 다르면 업데이트한다.
}
```

## ⏱️ 각 요청별 Timeout 추가

Cloudflare API, Public IP API등 외부 API를 요청할 때, go context기반의 Timeout제한을 추가했다. \
이제 요청 응답을 무기한 기다리려 하지 않게 되고, `--timeout`으로 정해진 초만큼만 각 외부 API요청을 기다린다. (기본값 2)

```go
// internal/cloudflare/client.go

// ...
func (c *Client) ListARecords(ctx context.Context, name string) ([]ARecord, error) {
	ctx, cancel := context.WithTimeout(ctx, c.timeout)
	defer cancel()
    // ...
}
// ...

// internal/publicip/client.go
// ...
// JSONClient도 동일하게 Timeout context를 생성
func (c PlainClient) Resolve(ctx context.Context, timeout time.Duration) (netip.Addr, error) {
	client := &http.Client{}
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()
    // ...
}
// ...
```


## ✅ 테스트 코드 추가

외부 서버에 요청을 날리는 경우, `httptest`를 이용하여 임의의 핸들러를 만들어 모킹할 수 있다. \
이를 기반으로 Public IP를 응답받았을 때 잘 저장하는지 등을 테스트한다.
```go
// ex. internal/publicip/client_test.go
func TestPlainClient_Resolve(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_, _ = fmt.Fprint(w, "192.0.2.10")
	}))
	defer server.Close()

	client := PlainClient{endpoint: server.URL}
	got, err := client.Resolve(context.Background(), time.Second)
	if err != nil {
		t.Fatalf("plain Resolve() Error: %v", err)
	}

	if got != netip.MustParseAddr("192.0.2.10") {
		t.Fatalf("plain Resolve(): got %v", got)
	}
}
```

## 🐳 컨테이너화

멀티스테이지 빌드를 통해서 바이너리 용량을 줄였고, HTTPS요청을 위해 ca-certificates만 추가 패키지로 받았다. \
nonroot로 실행하기 위해, `cfddns`라는 유저로서 실행되도록 하였다. \
이제는 컨테이너화를 통해 docker / kubernetes레벨에서 주기적으로 실행할 수 있게 되었다. \
그러나 여전히 Systemd 또는 Cron, Launchd에서도 사용가능하다.

```docker
FROM golang:1.25.12-bookworm AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -o /out/cfddns ./cmd/cfddns

FROM alpine:3.23.5

RUN adduser -D -H -u 10001 cfddns \
    && apk add --no-cache ca-certificates
COPY --from=builder /out/cfddns /usr/local/bin/cfddns

USER cfddns
ENTRYPOINT ["/usr/local/bin/cfddns"]
```

## 🧪 코드 검증: Pre-commit

Pre-commit을 이용하여 코드가 실제 커밋되기 이전에 여러 hook을 실행시켜서 코드를 검증시킨다. \
기본적인 formatting과 linting을 돌린 뒤, `go vet`으로 정적검사를 한다. \
커밋 메시지를 기록하려 할때, 커밋 메시지를 평가하여 컨벤션을 맞춘 커밋 메시지인지 확인하고, 실패하는 경우, 커밋하지 못하게 된다.

`go test`는 푸시 이전에 실행되도록 하였다. \
오픈소스로 배포된 Go 프로젝트를 위한 hook들이 이미 많이 존재하지만, 단순함을 위해 local에서 기본 도구인 `go vet`과 `go test`등을 실행하는 방식을 선택했다.

```yaml
default_install_hook_types:
  - pre-commit
  - pre-push
  - commit-msg

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict

  - repo: https://github.com/golangci/golangci-lint
    hooks:
      - id: golangci-lint-full
        args: [./...]

  - repo: local
    hooks:
      - id: go-vet
        entry: go vet ./...
      - id: go-test
        entry: go test --cover ./...

  - repo: https://github.com/compilerla/conventional-pre-commit
    hooks:
      - id: conventional-pre-commit
```

## 🚀 GitHub Actions

### CI: PR에서의 코드 검증

우선, `main`등의 브랜치에 병합되기 위해, 레포지토리 내의 PR에서 다음을 실행시킨다:
```yaml
name: Validate code before building artifacts

on:
  pull_request:

jobs:
  validate:
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod

      - run: go mod download
      - run: python -m pip install pre-commit
      - run: make check
      - run: go install golang.org/x/vuln/cmd/govulncheck@v1.6.0
      - run: $(go env GOPATH)/bin/govulncheck ./...

      - uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: fs
          scanners: vuln,secret,misconfig
          severity: HIGH,CRITICAL
          exit-code: 1
```

요약하자면, `pre-commit`의 내용들을 한번 더 실행시키고, `govulncheck`를 통해 소스 코드의 보안 취약점을 정적분석 한다. \
이후, Trivy를 이용해서 repository수준의 보안 취약점 검사를 진행한다. `HIGH` 또는 `CRITICAL`이상의 등급이 나오면 실패하게 된다. \
`pre-commit`이 별도 Actions로 추가될 수 있는 것으로 아는데, 단순함을 위해 그냥 직접 워크플로우에서 pre-commit을 설치하고 실행하도록 하였다. 

### 컨테이너 및 바이너리 배포

이제 `main`에 병합되거나, `v`로 시작하는 tag가 push된다면, 아래의 워크플로우가 동작한다. \
컨테이너 이미지를 빌드하고, 스캔한 뒤, 문제가 없으면 `ghcr`에 업로드한다. \
초반에 `amd64`만 빌드하고 검사를 돌리는 이유는, 현재 규모의 애플리케이션에서 단일 Runner의 상황에서 충분하다고 생각했기 때문이다. \
기껏 추가하는 패키지도 `ca-certificates`정도이니 아키텍처는 다르더라도 큰 문제가 일어날 것이라고는 생각하지 않았다. 

검사 이후, amd64, arm64모두 빌드하여 업로드한다. \
컨테이너 이미지가 무사히 업로드되면, Tag 기반 푸시인 경우, 단일 Runner에서 여러 아키텍처 및 OS에 맞게 바이너리를 빌드한다. \
이후, `gh`명령어를 이용해서 release를 게시시킨다.


```yaml
name: Release

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  build_image:
    steps:
      - uses: docker/build-push-action@v7
        with:
          platforms: linux/amd64
          load: true

      - uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: cf-ddns-client:scan
          severity: CRITICAL,HIGH
          exit-code: 1

      - uses: docker/build-push-action@v7
        with:
          platforms: linux/amd64,linux/arm64
          push: true

  build_binaries:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build_image
    steps:
      - run: |
          for target in linux/amd64 linux/arm64 darwin/amd64 darwin/arm64 windows/amd64; do
            os="${target%/*}"
            arch="${target#*/}"
            CGO_ENABLED=0 GOOS="${os}" GOARCH="${arch}" go build -trimpath ./cmd/cfddns
            # archive each binary into dist/
          done
      - run: gh release upload "${GITHUB_REF_NAME}" dist/* --clobber
```

## 🛏️ 후기

비록 간단한 Go 애플리케이션이지만, 실제로 내가 필요한 프로그램을 구조화하여 개발하니 뿌듯했고, 앞으로 나에게 필요한 일부 자동화 및 바이너리들을 더 많이 개발해볼 것 같다. \
특히, 릴리즈 게시까지의 과정은 처음 경험해서 매우 흥미로운 과정이었다. \
이 프로그램은 내가 필요해서 쓰는 것이기도 하고 지속적으로 업데이트를 할 수 있도록 할 것이다. \
단순한 코드로 유지해오던 도구였는데, 제대로 된 작은 프로젝트로 관리하니 더 많은 기능을 가지고 신뢰가능한 도구로 발돋움한 것 같다.


## 📚 Repository

[GitHub](https://github.com/fudoge/cf-ddns-client)
