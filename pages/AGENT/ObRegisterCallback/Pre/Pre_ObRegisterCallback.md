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

---

## 기술적인 설명

<br>

먼저 Pre 콜백함수에서 호출이 되었을 때, 인자의 OperationInformation->ObjectType 필드의 값이 **PsProcessType**인지 검증해야합니다.<br>



```c
  if (OperationInformation->ObjectType == *PsProcessType) {
     PEPROCESS PRE_eprocess = OperationInformation->Object;
     HANDLE PID = PsGetProcessId(PRE_eprocess);
```

검증을 할 때, OperationInformation->Object 값은 **EPROCESS**의 포인터 구조체가 됩니다.<br>

그런다음에는 여기서 현재 메모리에 로드된 프로그램을 식별하기 위하여 **PsGetProcessID** Native API를 이용하여 PID를 추출해야합니다.<br>

이제 우리에게는 PID가 있습니다. <br>

하지만,, 데이터베이스에서의 보호조치는 PID값이 존재하지 않습니다. 왜냐하면 PID는 메모리에 로드된 경우에만 유효하기 때문입니다.<br>

그래서 PID를 통하여 Binary를 구하고 최종적으로 **SHA256**문자열을 취득해야합니다. <br>
