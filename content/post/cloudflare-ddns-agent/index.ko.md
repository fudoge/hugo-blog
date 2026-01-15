---
title: "Cloudflare API를 이용한 Cloudflare DDNS Agent 개발"
description: Cloudflare API를 통해서 DDNS Agent를 개발해보자
date: 2026-01-15T21:15:30+09:00
image: cloudflare-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - HomeServer
    - Cloudflare
    - Golang
categories:
    - HomeServer
---

홈서버의 IP는 고정으로 가지기에 매우 힘들기에, DDNS를 세팅해줘야 한다.  
현재 Cloudflare에서 도메인을 구매하여 DNS서비스를 이용하고 있는데,  
[Cloudflare에서의 DDNS 구축 설명](https://developers.cloudflare.com/dns/manage-dns-records/how-to/managing-dynamic-ip-addresses/?_gl=1*136xy6p*_gcl_au*MTExOTI5NzA0Ni4xNzY1NTQyODIy*_ga*OTQ3YzNhMzMtNjA0Ny00ODI1LTlhYTQtYTFhN2MzYmQxY2Mz*_ga_SQCRB0TXZW*czE3Njg0Nzk0ODUkbzckZzEkdDE3Njg0Nzk0ODgkajU3JGwwJGgwJGRYZE5TeDN5dkpTeDhIRGZyN0kyaEJuN3lYRzhpLXVadjZB)에서는 Cloudflare API를 통해 자체개발하거나, [DDClient](https://github.com/ddclient/ddclient)를 이용하라고 한다.

직접 Cloudflare API를 Go 언어를 통해서 개발해보려고 한다.

---
## 📦 Cloudflare API 의존성 설치
Cloudflare API를 위한 코드를 받아준다.
```bash
go get github.com/cloudflare/cloudflare-go/v6
```

---
## ⚙️ 환경변수 설정
필요한 환경변수들은 아래와 같다:
```bash
CF_API_TOKEN        # Cloudflare API token (DNS_READ, DNS_WRITE requried)
ZONE_ID             # Cloudflare zone ID
DOMAIN_NAME         # DNS record name to update (e.g. home.example.com)
```

---
## 💻 코드
전체 코드는 아래와 같다:
```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"

	"github.com/cloudflare/cloudflare-go/v6"
	"github.com/cloudflare/cloudflare-go/v6/dns"
	"github.com/cloudflare/cloudflare-go/v6/option"
)

func main() {
	// Get IP..
	ip := getIP()
	name := os.Getenv("DOMAIN_NAME")
	zoneID := os.Getenv("ZONE_ID")

	client := cloudflare.NewClient(
		option.WithAPIToken(os.Getenv("CF_API_TOKEN")),
	)
	page, err := client.DNS.Records.List(context.TODO(), dns.RecordListParams{
		ZoneID: cloudflare.F(zoneID),
	})
	if err != nil {
		panic(err.Error())
	}

	res := page.Result
	for i := range res {
		if res[i].Name == name {
			if res[i].Content == ip {
				fmt.Println("DDNS record is up to date.")
				return
			} else {
				fmt.Printf("Updating DDNS record: A %v -> %v\n", res[i].Content, ip)
				updateRecord(client, zoneID, res[i].ID, name, ip)
				return
			}
		}
	}

	fmt.Printf("No such domain name: %v\n", name)
	fmt.Printf("Creating New Record...")
	createRecord(client, zoneID, name, ip)
}

func getIP() string {
	ipProvider := "https://api.ipify.org"
	res, err := http.Get(ipProvider)
	if err != nil {
		log.Fatalf("Error while requesting Public IP to %v: %v", ipProvider, err)
	}
	defer res.Body.Close()
	ip, err := io.ReadAll(res.Body)
	if err != nil {
		log.Fatalf("Error while reading response data from %v:  %v", ipProvider, err)
	}

	return string(ip)
}

func updateRecord(client *cloudflare.Client, zoneID, dnsRecordID, name, newIP string) {
	recordResponse, err := client.DNS.Records.Edit(
		context.TODO(),
		dnsRecordID,
		dns.RecordEditParams{
			ZoneID: cloudflare.F(zoneID),
			Body: dns.ARecordParam{
				Name:    cloudflare.F(name),
				TTL:     cloudflare.F(dns.TTL1),
				Type:    cloudflare.F(dns.ARecordTypeA),
				Content: cloudflare.F(newIP),
			},
		},
	)
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("%+v\n", recordResponse)
}

func createRecord(client *cloudflare.Client, zoneID, name, ip string) {
	recordResponse, err := client.DNS.Records.New(context.TODO(), dns.RecordNewParams{
		ZoneID: cloudflare.F(zoneID),
		Body: dns.ARecordParam{
			Name:    cloudflare.F(name),
			TTL:     cloudflare.F(dns.TTL1),
			Type:    cloudflare.F(dns.ARecordTypeA),
			Content: cloudflare.F(ip),
		},
	})
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("%+v\n", recordResponse)
}
```

### IP 가져오기
[ipify API](https://ipify.org)를 통해서 Public IP주소를 쉽게 가져올 수 있다.

```go
func main() {
	// Get IP..
	ip := getIP()
// ...
}

// ...

func getIP() string {
	ipProvider := "https://api.ipify.org"
	res, err := http.Get(ipProvider)
	if err != nil {
		log.Fatalf("Error while requesting Public IP to %v: %v", ipProvider, err)
	}
	defer res.Body.Close()
	ip, err := io.ReadAll(res.Body)
	if err != nil {
		log.Fatalf("Error while reading response data from %v:  %v", ipProvider, err)
	}
	return string(ip)
}
```


### `main.go` 주요 로직
퍼블릭 IPv4주소를 가져온 뒤, Cloudflare Client를 생성한다.  
그 뒤, 도메인에 대한 DNS 레코드들을 가져온다.

각 레코드에 대해서, 이 서버가 광고하길 원하는 이름을 찾는다.    
있다면, 업데이트를 시도하고, 없다면, 새로 만들기를 시도한다.

```go
func main() {
// ...
	name := os.Getenv("DOMAIN_NAME")
	zoneID := os.Getenv("ZONE_ID")

	client := cloudflare.NewClient(
		option.WithAPIToken(os.Getenv("CF_API_TOKEN")),
	)
	page, err := client.DNS.Records.List(context.TODO(), dns.RecordListParams{
		ZoneID: cloudflare.F(zoneID),
	})
	if err != nil {
		panic(err.Error())
	}

	res := page.Result
	for i := range res {
		if res[i].Name == name {
			if res[i].Content == ip {
				fmt.Println("DDNS record is up to date.")
				return
			} else {
				fmt.Printf("Updating DDNS record: A %v -> %v\n", res[i].Content, ip)
				updateRecord(client, zoneID, res[i].ID, name, ip)
				return
			}
		}
	}

	fmt.Printf("No such domain name: %v\n", name)
	fmt.Printf("Creating New Record...")
	createRecord(client, zoneID, name, ip)
}
```

### 레코드 업데이트

레코드를 업데이트하는 코드이다.  
`cloudflare.F()`는 Cloudflare SDK에서의 제네릭 헬퍼이다.  

```go
func updateRecord(client *cloudflare.Client, zoneID, dnsRecordID, name, newIP string) {
	recordResponse, err := client.DNS.Records.Edit(
		context.TODO(),
		dnsRecordID,
		dns.RecordEditParams{
			ZoneID: cloudflare.F(zoneID),
			Body: dns.ARecordParam{
				Name:    cloudflare.F(name),
				TTL:     cloudflare.F(dns.TTL1),         // Auto-TTL
				Type:    cloudflare.F(dns.ARecordTypeA), // A Record
				Content: cloudflare.F(newIP),
			},
		},
	)
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("%+v\n", recordResponse)
}
```

### 레코드 생성
`New()`로 새로운 레코드 생성 요청을 할 수 있다.
```go
func createRecord(client *cloudflare.Client, zoneID, name, ip string) {
	recordResponse, err := client.DNS.Records.New(context.TODO(), dns.RecordNewParams{
		ZoneID: cloudflare.F(zoneID),
		Body: dns.ARecordParam{
			Name:    cloudflare.F(name),
			TTL:     cloudflare.F(dns.TTL1),
			Type:    cloudflare.F(dns.ARecordTypeA),
			Content: cloudflare.F(ip),
		},
	})
	if err != nil {
		panic(err.Error())
	}
	fmt.Printf("%+v\n", recordResponse)
}
```

---
## 🏗️ 빌드
이제, 프로그램을 빌드한다.
```bash
go build -o ddns-agent main.go
chmod +x ddns-agent
sudo mv ddns-agent /usr/local/bin/ddns-agent
```

---
## 🔄 자동으로 동작하게 하기
광고하길 원하는 서버에서 주기적으로 동작시켜야 한다.  
Cron을 이용할 수 있지만, 더 깔끔한 관리를 위해 Systemd가 관리하도록 해보자.  


### Linux(Systemd)
`/etc/systemd/system/ddns.service`파일을 생성해서, 아래와 같이 작성한다.  
`Environment=<key>=<value>`에서 환경변수 값을 채워주자.

```ini
# /etc/systemd/system/ddns.service
[Unit]
Description=DDNS Agent (Cloudflare)
# Execute when network service is available
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot # Job-like service
ExecStart=/usr/local/bin/ddns-agent # Execute binary
# Environment variables
Environment=CF_API_TOKEN=xxxx
Environment=ZONE_ID=xxxx
Environment=DOMAIN_NAME=xxxx

## Security
# No privilege escalation
NoNewPrivileges=true
# Isolate tmpfs
PrivateTmp=true
# ReadOnly System directories(/etc, /usr, ...)
ProtectSystem=strict
# Cannot Access /home
ProtectHome=true

[Install]
WantedBy=multi-user.target # Multi-user unit
```

`/etc/systemd/system/ddns.timer`파일을 만들어서, 주기적으로 동작하는 트리거를 만든다.
```ini
# /etc/systemd/system/ddns.timer
[Unit]
Description=Run DDNS Agent periodically

[Timer]
OnBootSec=30 # First execution: 30s after boot
OnUnitActiveSec=5min # 5-min period
Persistent=true # Ensures executed once after inactivate time

[Install]
WantedBy=timers.target # Timer unit
```

아래 명령으로 적용한다.
```bash
sudo systemctl daemon-reload
sudo systemctl enable ddns.timer
sudo systemctl start ddns.timer
```

잘 등록되어있는지 확인한다.
```bash
systemctl list-timers | grep ddns
```

### MacOS(launchd)
MacOS를 쓰는 경우, Systemd 대신, Launchd를 쓴다.  

사용자 로그인세션부터가 아닌 부팅시점부터 도리고 싶다면, `LaunchAgents` 대신 `LaunchDaemons`를 써야 한다.

`yourname`에는 홈 유저네임을 쓰면 된다.

```xml
# ~/Library/LaunchAgents/com.yourname.ddns-agent.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.yourname.ddns-agent</string>

    <!-- Program -->
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/ddns-agent</string>
    </array>

    <!-- EnvironmentVariables key and its dictionary -->
    <key>EnvironmentVariables</key>
    <dict>
        <key>CF_API_TOKEN</key>
        <string>YOUR_CF_API_TOKEN</string>
        <key>ZONE_ID</key>
        <string>YOUR_CF_ZONE_ID</string>
        <key>DOMAIN_NAME</key>
        <string>YOUR_DDNS_RECORD_NAME</string>
    </dict>

    <!-- AutoRun when Startup -->
    <key>RunAtLoad</key>
    <true/>

    <key>StartInterval</key>
    <integer>300</integer> <!-- 5 mins interval-->

    <!-- stdout/stderr -->
    <key>StandardOutPath</key>
    <string>/usr/local/var/log/ddns.log</string>
    <key>StandardErrorPath</key>
    <string>/usr/local/var/log/ddns.err</string>
</dict>
</plist>
```

아래 명령으로 적용한다.
```bash
mkdir -p /usr/local/var/log # 로그 존재하도록 정리
launchctl load ~/Library/LaunchAgents/com.yourname.ddns-agent.plist
launchctl start com.yourname.ddns-agent
```

잘 등록되어있는지 확인한다:
```bash
launchctl list | grep ddns
```

---
## 📚 References
- [Cloudflare API(Go)](https://developers.cloudflare.com/api/go/resources/dns/)