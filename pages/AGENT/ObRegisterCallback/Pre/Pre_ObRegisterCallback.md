---
layout: default
title: Pre__ObRegisterCallback_info
nav_order: 1
permalink: /pages/AGENT/ObRegisterCallback/Pre
grand_parent: AGENT
parent: ObRegisterCallback_info
typora-root-url: ../../../../
---

# **Pre_ObRegisterCallback 콜백함수에 대하여 설명합니다.**

### 이것이 진짜  Thread, Process가 실행/복제/삭제등이 일어날 때 호출됩니다. 이 콜백함수가 실행될 때 보호조치를 수행합니다.

<br>

이 함수가 호출되면, 다음과 같은 과정을 거칩니다.

<br>

1. 프로세스의 PID를 얻습니다.
2. 유틸리티를 통하여 PID값으로 SHA256값을 구합니다.
3. 전역변수에 저장된 보호조치를 위한 **연결리스트**를 구한 SHA256 필터링하여 일치할 때까지 순환하여 찾습니다. 
4. 일치한 PID일 때, 보호조치 적용대상의 타입인지 확인합니다.
5. 타입이 옮다면 DesiredAccess 에서 권한을 제거합니다.
