---
layout: default
title: ObRegisterCallback_info
nav_order: 4
permalink: /pages/AGENT/ObRegisterCallback
parent: AGENT
has_children: true
typora-root-url: ../../../
---

# **ObRegisterCallback 콜백함수에 대하여 설명합니다.**

### 이것은 주로 Thread, Process가 실행/복제/삭제등이 일어날 때 호출되며, NotifyRoutine과 다르게 이 함수에서는 **보호조치(차단)**를 시행합니다. 

ObRegisterCallback 콜백함수는 모니터링을 하지않고, 데이터베이스에 저장된 **프로그램의 SHA256**값과 **보호조치 타입**에 따라 보호조치를 시행합니다. <br>

주로 악성코드가 AI_Server로부터 판단되었을 때나, 관리자에 의해 프로그램의 SHA256을 기준으로 보호조치를 할 수 있게 구현되었습니다.  <br>

이들이 AGENT_ID를 통해 식별된 에이전트에서 특정 프로그램의 실행,제거등을 차단(Block)하려면 데이터베이스에 해당 정보를 저장해야합니다. 그렇게 하기 위해서는 중앙서버의 명령을 호출하여 대신 정보를 저장하게 합니다. <br>

정보 저장이 성공된 경우, 즉시 **중앙서버는 AGENT_ID를 통하여 에이전트에게 보호조치 명령을 요청**하고 에이전트는 처리 결과를 위해 응답합니다. <br>

자세한 보호조치는 하위 디렉터리에서 다룹니다.



