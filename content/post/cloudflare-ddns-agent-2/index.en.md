---
title: "Improving My Cloudflare DDNS Client"
description: "Improving the Cloudflare DDNS Agent I built earlier into a more structured DNS record automation tool"
date: 2026-08-03T20:21:44+09:00
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

## ℹ️ Overview

In a [previous post](/ko/p/cloudflare-api를-이용한-cloudflare-ddns-agent-개발/), I used the Cloudflare API and Golang to build a small program that automated DDNS record updates. \
That previous post is currently available only in Korean. \
However, that version had limited options, and I also wanted to clean up the code structure.

Since I recently started reorganizing my homelab, I decided to turn the simple Go program for updating Cloudflare DDNS records into something closer to a proper application.

## 📁 Folder Structure

```text
.
|-- cmd
|   `-- cfddns
|       `-- main.go
|-- internal
|   |-- cloudflare
|   |   `-- client.go
|   |-- config
|   |   |-- config.go
|   |   `-- config_test.go
|   |-- ddns
|   |   |-- sync.go
|   |   `-- sync_test.go
|   `-- publicip
|       |-- client.go
|       `-- client_test.go
|-- Dockerfile
|-- go.mod
|-- go.sum
|-- LICENSE
|-- Makefile
`-- README.md
```

The previous version only had a single `main.go` file in the repository root. This time, I reorganized it into a more typical Golang project layout. \
The executable entry point is placed under `cmd/`, and the internal logic is placed under `internal/`.

Each package is responsible for the following:

- `internal/cloudflare`: Cloudflare API wrapper
- `internal/config`: loading flags and environment variables
- `internal/ddns`: the actual mode-specific DDNS logic
- `internal/publicip`: loading the public IP address

I also added tests to verify the code.

## 🌐 Flexible Public IP Providers

Previously, the program could only fetch the public IP address from Ipify. Now it supports more options. \
If Ipify has an issue, I can now switch to another provider. \
I improved it so that two types of public IP responses can be used:

- Plain text response: the response body is just the IP address
- JSON response: the response body is returned as JSON

When a value such as `--jsonpath "$.ip"` is passed to the CLI, the program uses `JSONClient`. If that flag is not used, it defaults to `PlainClient`. \
If `--endpoint` is not specified, the program requests the public IP from Ipify by default.

Usage examples:

```bash
# Plain text response, compatible with the default ipify endpoint.
cfddns --endpoint https://api.ipify.org

# JSON response.
cfddns --endpoint 'https://api64.ipify.org?format=json' --jsonpath '$.ip'
```

Both the `PlainClient` and `JSONClient` structs implement the `IPResolver` interface.

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

## ⚙️ Added Mode Support

Previously, the program simply looked up the current record, updated it if it existed, and created a new one if it did not.

For now, I only use `replace` myself. Still, I thought `append` mode could be useful for someone running a homelab with multiple public IPs under the same domain name, especially if they want to distribute traffic at the DNS level.

Now it uses more fine-grained logic:

- `replace` mode: keeps only the current public IP address. This is the default behavior.
- `append` mode: allows the domain name to have multiple records.

The TTL value is also configurable now instead of always using the default. Even if an existing record already has the same IP address, the program updates it when the TTL is different.

```go
// internal/ddns/sync.go
type DNSClient interface {
	ListARecords(ctx context.Context, name string) ([]cloudflare.ARecord, error)
	CreateARecord(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error
	UpdateARecord(ctx context.Context, recordID, name string, ip netip.Addr, ttl dns.TTL) error
	DeleteARecord(ctx context.Context, recordID string) error
}

func (s *Syncer) Append(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error {
	// If an A record with the same IP exists, only fix the TTL.
	// Otherwise, create a new A record.
}

func (s *Syncer) Replace(ctx context.Context, name string, ip netip.Addr, ttl dns.TTL) error {
	// Delete A records that do not match the current public IP.
	// If multiple records have the same IP, keep only one.
	// Update TTL when the remaining record has a different value.
}
```

## ⏱️ Added Timeouts for Each Request

I added Go context-based timeout limits when calling external APIs such as the Cloudflare API and the public IP API. \
The program no longer waits forever for responses. Instead, each external API request waits only for the number of seconds configured with `--timeout`. The default is `2`.

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
// JSONClient creates a timeout context in the same way.
func (c PlainClient) Resolve(ctx context.Context, timeout time.Duration) (netip.Addr, error) {
	client := &http.Client{}
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()
    // ...
}
// ...
```

## ✅ Added Tests

When a function makes requests to an external server, `httptest` can be used to create a mock handler. \
Based on that, I tested whether the client correctly stores the returned public IP address.

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
## 🐳 Containerization

I reduced the binary size with a multi-stage build, and only added `ca-certificates` as an extra package for HTTPS requests. \
To run the container as a non-root user, I configured it to run as a user named `cfddns`.

```dockerfile
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

## 🧪 Code Validation: Pre-commit

I use pre-commit to run several hooks before code is actually committed. \
After basic formatting and linting, it runs static checks with `go vet`. \
When a commit message is created, it also checks whether the message follows the convention. If it does not, the commit fails.

`go test` runs before pushing. \
There are already many open-source hooks for Go projects, but for simplicity I chose to run the basic local tools such as `go vet` and `go test` directly.

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

### CI: Code Validation on PRs

Before a PR can be merged into a branch such as `main`, the repository runs the following workflow:

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

In short, the workflow runs the same checks from `pre-commit` again, then uses `govulncheck` to statically analyze security vulnerabilities in the source code. \
After that, Trivy scans the repository for security issues. If it finds anything with `HIGH` or `CRITICAL` severity, the workflow fails. \
I know pre-commit can be added as a separate GitHub Action, but for simplicity I chose to install and run pre-commit directly in the workflow.

### Container and Binary Releases

Now, when a change is merged into `main` or a tag starting with `v` is pushed, the workflow below runs. \
It builds the container image, scans it, and uploads it to GHCR if there are no issues. \
The reason I first build and scan only `amd64` is that, for an application of this size on a single runner, that is enough for my current use case. \
The only extra package added to the image is `ca-certificates`, so I do not expect architecture-specific issues to be significant.

After the scan, the workflow builds and uploads both `amd64` and `arm64` images. \
Once the container image is uploaded successfully, tag-based pushes also build binaries for multiple architectures and operating systems on a single runner. \
Finally, it publishes a release using the `gh` command.

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

## 🛏️ Closing Thoughts

Even though this is a small Go application, it felt good to structure and develop a program that I actually need. I think I will keep building more automation tools and small binaries for my own use. \
The release publishing flow was especially interesting because it was my first time setting that up end to end. \
Since this is a tool I use myself, I plan to keep updating it continuously. \
It used to be a tool maintained as simple code, but managing it as a proper small project makes it feel like it has grown into a more capable and reliable tool.

## 📚 Repository

[GitHub](https://github.com/fudoge/cf-ddns-client)
