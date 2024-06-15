---
layout: default
title: AGENT_IOCTL
nav_order: 5
permalink: /pages/AGENT/IOCTL
parent: AGENT
typora-root-url: ../../../
---

# **유저모드와 커널모드간 통신에 대하여 설명합니다.**

### 유저모드에서 Windows Kernel Driver를 Load하고, IOCTL를 통하여 필요한 정보를 버퍼를 주고받습니다. 

---



유저모드 프로그램은 에이전트 커널 드라이버를 로드하고, IOCTL를 통하여 유저모드에서 자신이 준비한 Buffer를 전달합니다. <br>

이때, 유저모드 프로그램은 커널로부터 응답을 받을 때까지 종료할 수 없으며, 대기상태가 됩니다. <br>

커널에서 중앙서버(RUST)와 통신하여 얻은 결과 및 예기치 못한 커널 동작에 의한 결과들을 유저모드 프로그램에 전달하여 응답하는 형식입니다. <br>

응답을 받은 유저모드 프로그램은 마지막 과정을 거치고 종료됩니다.



---

## 기술적인 설명
