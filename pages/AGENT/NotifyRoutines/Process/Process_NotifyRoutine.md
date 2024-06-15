---
layout: default
title: Process_NotifyRoutine
nav_order: 1
permalink: /pages/AGENT/NotifyRoutines/Process
parent: NotifyRoutine_info
grand_parent: AGENT
typora-root-url: ../../../../
---

# **프로세스 콜백함수에 대하여 설명합니다.**

### 이것은 커널에서 PsSetCreateProcessNotifyRoutineEX() 콜백함수를 이용하여 데이터를 수집하여 모니터링합니다.

---

DriverEntry()함수에서 실행될 때 최초 등록해야합니다. <br>

등록이 성공되면, PsSetCreateProcessNotifyRoutineEx()에 등록할 때 가르킨 함수의 주소에서 콜백함수 호출이 일어납니다.<br>

이때는 다음과 같은 정보를 저장합니다. 



- 프로세스 생성여부 (char)
- 프로세스 ID (handle)
- 프로세스 프로그램의 절대경로 (char(UNICODE to ANSI))
- 프로세스 프로그램의 사이즈 (ULONG32)
- 프로세스 프로그램의 SHA256 (64byte)
