---
title: "CKAD(Certified Kubernetes Application Developer) 후기 (2026.06)"
slug: ckad-review-2026
description: CKAD 응시 및 합격후기(2026년 6월)와 준비 과정, 최신 정보, 팁.
date: 2026-06-18T10:49:11+09:00
image: certification.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - Journey
categories:
    - Certification
---


작년 말에 프로그래머스에서 얼리버드 할인으로 약 27~8만원에 바우처를 구매해서, 시험을 보게 되었다.

---
## ℹ️ 시험 정보

**CKAD(Certified Kubernetes Application Developer)**
- **주관:** Linux Foundation
- **시험 내용 및 범위:** Kubernetes에서의 애플리케이션 배포에 관련한 Performance-based 시험
    - Application Design and Build: 20%
    - Application Deployment: 20%
    - Application Observability and Maintenance: 15%
    - Application Environment, Configuration and Security: 25%
    - Services and Networking: 20%
- **시험 시간:** 120분
- **Kubernetes 버전:** v1.35 (2026. 06. 17기준, 보통 마이너 버전이 출시되고 1~2개월 이후에 시험버전이 업데이트)
- **비용:** $445(2026.06 기준)
    - 구매 시, 1년 이내로 시험을 신청할 수 있음
    - 1번의 재기회 제공
    - 정가에 구매하지 말고, 쿠폰을 통한 할인을 노리는 것 권장
- **시험 환경:** 온라인, PSI Secure browser 사용, Xfce 기반 Linux 호스트에서 문제별 SSH접속
    - 자세한건 [Linux Foundtation](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad)에서 확인
    - 또는 [본 블로그에서의 CKA 후기]( {{<relref "post/cka-review-2026" >}} ) 참고

---
## 📖 시험 준비

총 준비 기간은 시험 당일 포함 3일이다. \
CKA때 대비 짧은 기간인데, CKA경험이 있어서 자신있기도 했고, 이번에 선택한 강의는 좀더 시험에 최적화된 강의여서 그렇다. \
강의는 [인프런 - CKAD Practical Exam Guide](https://www.inflearn.com/course/certified-kubernetes-2?cid=341153)강의를 들었다. 

1일차에는 실습과 함께 강의를 쭉 달렸다. \
2일차에는 Killer.sh모의고사를 보면서 환경에 적응헸는데, \
Killer.sh 문제는 시험 범위에서 벗어난 게 일부 있는 것 같았고, 난이도가 있었지만, 통과는 하였다. \
이후, 다시 강의에서 다룬 연습문제들을 스스로 풀어봤다. \
이 때에는 강의에서 제공하는 virtualvox + vagrant기반 환경 대신, 내 홈랩 클러스터에 Traefik만 설치해서 연습했다.

시험 당일에는 아침에 스터디카페에 걸어가면서 정리한 노트들을 다시 읽었다.

---
## ✏️ 시험 응시

이전 CKA 응시때에도 스터디카페를 이용했는데, 이번에도 스터디카페에서 스터디룸을 미리 예약해서 시험을 봤다. \
스터디룸 예약을 09:00 ~ 12:00으로 잡고, 시험 예약을 09:30 ~ 11:30으로 해놓으니 딱 적절했다.

시험 환경에는 30분 전부터 입장가능해서 미리 들어가 있으면 일찍 주변 환경 점검을 할 수 있고, 바로 시험을 볼 수 있다. \
입실 시작부터 주변 환경을 보여주는데, 책상 아래까지 꼼꼼히 노트북 웹캠으로 보여줬다. \
감독관마다 기준이 조금씩 다른데, CKA때와 똑같은 스터디룸에서 진행했을 때에는 스터디룸의 CCTV에 대해 지적하지 않았지만, 이번에는 지적받아서 CCTV의 맞은편으로 방향을 바꿔 앉았다. \
스마트폰으로도 QR을 연결하면 시험 환경을 보여주는 데 잠깐 활용할 수 있다고 한다. \
마지막으로, 스마트폰을 치우는 모습을 보여주고, 손목과 귀를 돌리면서 아무 착용이 없는 것을 보여준 뒤, 시험을 시작했다. \
노트북 거치대와 무선마우스의 사용은 직접 물어봐서 허락받았다.

시험이 본격적으로 시작되고, 문제를 풀었다. \
강의 덕분에, 시험을 편안하게 볼 수 있었다. \
이미 CKA와 겹치는 부분이 많은 것도 한 몫을 한 것 같다. \
60분 동안 풀고, 시간이 너무 남아서 20분동안 검토했고, 40분이 남았을 때 조기종료를 했다.

문제는 자유롭게 돌아다닐 수 있으며, flag를 세우면 리스트에서 마킹되어 다시 검토하기 편하게 되어있다. \
문제의 맨 위에는 관련 문서 링크를 제안해주고, 접속해야 할 SSH 정보도 바로 복사-붙여넣기 가능하다. \
본문은 시나리오를 설명하는 context와 해야하는 일인 task로 나뉜다. 

vscodium이 사용가능하며, 플러그인은 설치할 수 없다. \
나는 그냥 터미널 + vim을 이용했는데, vsc기반의 환경을 선호한다면 사용하는 것도 좋아보인다.

---
## 🚀 시험 결과

![Exam result](result.png)

시험 시작 기준 약 24시간 뒤, 결과가 나왔다! \
점수는 94점을 받았다. 세부 채점 결과는 알 수 없다.

![Certification](certification.png)
자격증은 PDF로 다운받을 수 있으며, Credly를 통해 Open Badge가 제공된다.

---
## 🍕 최신 정보 (2026. 06) 및 시험에 대한 팁

CKA보다는 범위도 좁고 상대적으로 쉬운 시험인 것 같다. \
**다른 과거의 후기들을 보면, alias 및 자동완성 설정을 쉘에서 해놓으라고 하는데, 이제는 할 필요가 없다.** \
base host에서 ssh를 통해 문제마다 다른 환경에 옮겨다녀야 하며, `alias k="kubectl"`, 자동완성이 이미 세팅되어있다. \
과한 alias설정은 문제마다 환경이 옮겨지므로 거의 의미가 없다(ex. `kgpoy = kubectl get pod -o yaml`). \
그냥 기본 세팅대로 하면 된다.

이외 시험에 대한 팁은 여러 가지 있다:
- **문제를 잘 읽고, namespace혼동이 없게 하자. 사소한 표현 하나하나가 서브태스크이거나 조건일 수 있다.**
- **복사-붙여넣기를 적극 활용하자. 리소스 이름과 같은 것들을 복붙하여 오타를 줄일 수 있다.**
    - 브라우저는 `Ctrl + <C/V>`, 터미널은 `Ctrl + Shift + <C/V>`.

---
## 🛏️ 시험 후기 및 마무리

CKAD를 취득했지만, 여전히 부족하며, 앞으로 배워야 할 것들이 많다. \
다음은 **CKS** 의 바우처가 아직 있어서 이걸 여름방학 때 준비할 것 같고, \
보안 쪽 부분은 특히 더욱 중요해지고 있으며 나도 잘 모르는 부분이 많기에 자격증 준비 이상의 공부를 할 필요가 있을 것 같다.
