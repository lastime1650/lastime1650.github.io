---
layout: default
title: AGENT_Socket_Processing
nav_order: 1
permalink: /pages/AGENT/Socket/AGENT_Socket_processing
parent: AGENT_Socket
grand_parent: AGENT
typora-root-url: ../../../../
---

# **커널에서 중앙서버의 요청을 처리하는 방법을 설명합니다.**

### 에이전트와 중앙서버간 초기통신이 이루어지면, 에이전트는 항상 Receive상태가 됩니다.  그렇기 때문에 중앙서버의 요청이 있기 전까지 데이터를 전달하지 않는 형식입니다. <br>

<br>



{: .warning }

> 이 경고창이 보이는 경우, 이 파트는 완전히 정의가 이루어지지 않음을 뜻합니다.

<br>

먼저 중앙서버에서 에이전트에게 보내는 요청은 다음과 같습니다.



- 에이전트 활성화 여부확인
- 에이전트에게 SHA256하나 전달하여 바이너리 받기
- ObRegisterCallback에 적용할 보호조치 등록

<br>

{: .important-title }

>왜 수신받는 상태인가?
>
>
>
>서로 간 연결할 때 포트를 늘려 통신하지 않고, 단일 TCP 세션으로 데이터를 전송하는 방식으로 채택하였습니다. <br>
>
>또한 에이전트는 모니터링된 데이터를 전송하는 것만 하지 않고 중앙서버의 보호조치 명령을 받아 수행해야합니다.



