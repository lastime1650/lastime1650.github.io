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

하지만,, 데이터베이스에서의 보호조치 칼럼에는 PID값이 존재하지 않습니다. 왜냐하면 PID는 메모리에 로드된 경우에만 유효하기 때문입니다.<br>

그래서 PID를 통하여 Binary를 구하고 최종적으로 **SHA256**문자열을 취득해야합니다. <br>



```c

if (OperationInformation->Operation == OB_OPERATION_HANDLE_CREATE) {
    if (OperationInformation->Parameters->CreateHandleInformation.DesiredAccess & PROCESS_CREATE_PROCESS) {
        /* 프로세스 생성 시,,, */

        /*
            프로세스 생성 시 Block 일 때,
        */

        // Action 노드 추출
        if (Action_for_process_routine_node_Start_Address != NULL) {
            DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "Process PreOperationCallback: Handle = %p, %llu\n", OperationInformation->Object, PID);

            PAction_for_process_creation_NODE ActionNode = Get_Action_Node_With_SHA256(PID);// 하나의 노드를 가져오면 해당 프로그램의 프로세스에 대해 Action처리하면됨
            if (ActionNode) {
                DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "PRE _ ActionNode [Create] 찾음! - PID: %llu \n", PID);
                Action_PROCESSING_on_RegisterCallback(ActionNode, &OperationInformation, Create);
                DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "DesiredAccess -> %d \n\n", OperationInformation->Parameters->CreateHandleInformation.DesiredAccess);
            }

        }
    }
```

<br>

SHA256을 구하는 것은 파일 입출력을 사용하기 때문에, 무작정 구하고 보는 것은 추천하지 않았습니다. <br>

그전에 먼저 OperationInformation->Operation값을 통하여 프로세스가 생성과 관련한 명령이 확인하고, 세부적으로 프로세스의 행동을 감지하기 위해,<br>

`OperationInformation->Parameters->CreateHandleInformation.DesiredAccess`

Create일때의 DesiredAccess 권한에서 **(& PROCESS_CREATE_PROCESS )** AND연산하여 일치하면 이 프로세스는 실제로 생성중인 상태를 의미하도록 세부적인 처리가 가능하도록 구현하였습니다.<br>

그래서 먼저 커널내에서 전역변수에 저장된 **연결리스트**가 NULL인지,아닌지를 확인하고, <br>

PID를 인수로 넘기면, Binary를 구하고 연결리스트의 **Head**부분부터 SHA256필드를 조회하여 **일치한 노드**를 반환하는 **Get_Action_Node_With_SHA256()**를 호출하도록 구현하였습니다. <br>

여기서 반환값은 추출된 노드의 주소이거나 NULL입니다. 만약 여기서 NULL이 아닌 값이라면, 데이터베이스의 보호조치를 하라고 해석되므로, 추출된 노드의 필드에 저장된 값에 따라서 보호조치를 적용하라고 되어 있습니다. 
