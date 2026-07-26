---
title: "CKS(Certified Kubernetes Security Specialist) Review (2026.07)"
slug: cks-review-2026
description: CKS exam and passing review from July 2026, including preparation, latest notes, and tips.
date: 2026-07-26T09:19:34+09:00
image: certification.png
math: 
license: 
hidden: false
comments: true
draft: true

tags:
    - Kubernetes
    - Journey
categories:
    - Certification
---

Late last year, I bought the voucher through an early-bird discount on Programmers for about 270,000 to 280,000 KRW. \
I took the Kubernetes certification exams in the order of CKA -> CKAD -> CKS, and this was my third Kubernetes certification exam.

---
## ℹ️ Exam Information

**CKS(Certified Kubernetes Security Specialist)**
- **Organizer:** Linux Foundation
- **Exam scope:** Security best practices across the Kubernetes ecosystem
    - Cluster Setup: 15%
    - Cluster Hardening: 15%
    - System Hardening: 10%
    - Minimize Microservice Vulnerabilities: 20%
    - Supply Chain Security: 20%
    - Monitoring, Logging, and Runtime Security: 20%
- **Duration:** 120 minutes
- **Kubernetes version:** v1.35 (as of 2026.07.25; the exam version is usually updated a few weeks after a new minor release)
- **Cost:** $445 (as of 2026.07)
    - After purchase, you can schedule the exam within 1 year.
    - 1 free retake is included.
    - I recommend looking for discount coupons instead of paying the full price.
- **Exam environment:** Online, using PSI Secure Browser, with an Xfce-based Linux host and SSH access for each task
    - See the [Linux Foundation guide](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad) for details.
    - You can also refer to [my CKA review]( {{<relref "post/cka-review-2026" >}} ).
- **Prerequisite:** CKA certification
- **Benefit:** Automatically renews your CKA certification
    - [Source: Expanding CARE: Passing CKS can now extend your CKA certification](https://training.linuxfoundation.org/blog/expanding-care-passing-cks-can-now-extend-your-cka-certification/)
    - [CARE Program](https://training.linuxfoundation.org/care-program/)

---
## 📖 Preparation

My preparation period was about 3 weeks. \
For the lecture, I took [Udemy - Certified Kubernetes Security Specialist 2026 by Zeal Vora](https://www.udemy.com/course/certified-kubernetes-security-specialist-certification). The explanations were good, and the PDF and lab materials were also well provided.

Two days before the exam, I solved one Killer.sh mock exam and reviewed it. \
On the day before the exam, I solved and reviewed the second Killer.sh mock exam. \
Since I had already scored well on Killer.sh, I did not really think I would fail. \
On the morning of the exam, I read my weak-topic notes and wrong-answer notes on my phone while walking to the study cafe.

---
## ✏️ Taking the Exam

I used a study cafe for my previous CKA and CKAD exams, and this time I also reserved a study room at the same study cafe. \
I booked the room from 09:00 to 12:00 and scheduled the exam from 09:30 to 11:30, which was just about right.

You can enter the exam environment 30 minutes before the scheduled time, so if you join early, you can finish the room check early and start the exam right away. \
From the beginning of check-in, I had to show the surrounding environment, including the area under the desk, with my laptop webcam. \
Unlike my CKAD exam, I was also asked to verify my passport this time. The process seems to depend on the proctor, so it is better to keep it ready even if it is already uploaded to PSI. \
As with my CKAD exam, the proctor pointed out the CCTV in the room and told me to make sure it could not record my laptop screen, so I sat on the opposite side of the CCTV. \
Finally, I showed that I had put my smartphone away, rotated my wrists and ears to show that I was not wearing anything, and then started the exam.

After the exam started, I began solving the tasks. \
About half of the questions had a fair number of subtasks, while the rest could be solved quickly if you were prepared. \
I ended up spending a lot of time on a Falco question. Fortunately, I had solved the other questions first and handled it at the end, which helped.

You can move freely between questions, and if you flag a question, it is marked in the list, making it easy to revisit later. \
At the top of each question, related documentation links are suggested, and the SSH information you need to access the environment can be copied and pasted directly. \
Each question body is divided into a context section that describes the scenario and a task section that describes what you need to do.

VSCodium is available, but plugins cannot be installed. \
I just used the terminal and vim, but if you prefer a VS Code-based environment, it seems usable.

---
## 🚀 Result

![Exam result](result.png)

The result came out about 24 hours after the exam started. \
I scored 80. Detailed scoring information is not provided.

![Certification](certification.png)

The certificate can be downloaded as a PDF, and an Open Badge is provided through Credly.

---
## 🍕 Latest Notes (2026.07) and Exam Tips

As expected, CKS felt harder than CKA and CKAD. \
If I had to rate the difficulty, it would be roughly like this:
- CKAD: ★★★☆☆ (3/5)
- CKA:  ★★★⯨☆ (3.5/5)
- CKS:  ★★★★☆ (4/5)

CKS is not limited to Kubernetes resources. It also requires knowing more detailed conditions and options needed for hardening cluster configuration, and the exam scope itself is quite broad. \
You need to know a much wider range of topics, including BOM, Trivy, Istio, Cilium, and Falco. \
In my case, Cilium was not too much of a burden because I had used it many times before, but I had not properly used Istio or Falco. \
It also requires a higher level of Linux usage and practical command-line ability.

I also have a few general exam tips:
- **Read each question carefully and avoid namespace mistakes. Even small wording details can be subtasks or conditions.**
- **Use copy and paste actively. Copying resource names helps reduce typos.**
    - In the browser, use `Ctrl(Cmd) + <C/V>`. In the terminal, use `Ctrl + Shift + <C/V>`.
- To receive Falco output immediately instead of buffered output, use `falco -U` or `falco --unbuffered`.
- If you have time, always verify your answer directly. For example, after creating a NetworkPolicy or CiliumNetworkPolicy, test communication between Pods, or check whether a ServiceAccount token is properly mounted as a projected volume.

---
## 🛏️ Final Thoughts

I am glad that I earned the CKS certification, but honestly, I still need deeper experience with tools like Falco and Istio. \
I did get some exposure through exam preparation, but actually using and applying them properly in a cluster will have to be a next step. \
**Still, it is clear that preparing for CKS significantly improved my knowledge of security in the Kubernetes ecosystem and gave me a stronger foundation for building secure clusters.**

Recently, between CKS and TOEIC Speaking for graduation, there have been a lot of things going on, so I have not posted much on the blog. \
I plan to fill it with more notes and experiences going forward.
