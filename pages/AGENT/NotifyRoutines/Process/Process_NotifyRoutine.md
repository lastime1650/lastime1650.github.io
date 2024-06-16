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



그리고 이 정보는 각각 순차적으로 노드를 생성하여 연결리스트를 만듭니다. 이 연결리스트는 **전역변수에 "길이-기반"형태**로 저장되다가,중앙서버의 요청이 있기 전까지 보관되며, 장기간 방치된 경우 DISK에 저장됩니다.

---

## 기술적인 설명<br>

 먼저 모니터링하기 위해서 미리 구현된 핸들러 함수의 주소를 인수로 넘겨야합니다.

`status = PsSetCreateProcessNotifyRoutineEx(PcreateProcessNotifyRoutineEx, FALSE);`

코드에서는 **PcreateProcessNotifyRoutineEx**이름으로 핸들러를 구현하였습니다.<br>

`void PcreateProcessNotifyRoutineEx(PEPROCESS Process, HANDLE ProcessId, PPS_CREATE_NOTIFY_INFO CreateInfo)`

또한 핸들러는 Ex가 들어간 콜백함수이기 때문에 세부적인 프로세스를 식별할 수 있는 인자가 제공됩니다. <br>



핸들러에서 모니터링된 정보를 저장하기 전에 중앙서버와 연결되었는 지 확인하기 위하여 **is_connected_2_SERVER** 부울 전역변수의 값으로 검증합니다. <br>

```c
if (!is_connected_2_SERVER) {
	DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "PcreateProcessNotifyRoutineEx -> is_connected_2_SERVER 값: FALSE \n");
	return;

}
```

그리고 여기 프로세스를 통해서 얻는 다양한 정보를 담는 구조체가 있습니다.<br>

```c
typedef struct PROCESS_ROUTINE_struct {
	KEVENT event_for_saving;// 다른 스레드로 정보 이전용..  잠시 알림루틴함수 스레드의 흐름을 얼리기 위함

	PEPROCESS eprocess_addr;

	BOOLEAN is_process_creating; // TRUE면 프로세스 생성임을 인지
	HANDLE process_handle;//핸들 값
	HANDLE process_pid;//PID 값

	PUCHAR EXE_bin;
	ULONG EXE_bin_size;
	PUCHAR sha256;

	UNICODE_STRING process_ImageName;
}PROCESS_ROUTINE_struct, * PPROCESS_ROUTINE_struct;
```

이들은 필요에 따라 필드에 저장됩니다. <br>

하지만 이 구조체를 통째로 중앙서버에 전달하지 않습니다. 좀 더 과정을 설명해야합니다. <br>

```c
PROCESS_ROUTINE_struct TMP_data_pack = { 0 }; // 공통으로 사용되는 녀석
KeInitializeEvent(&TMP_data_pack.event_for_saving, SynchronizationEvent, FALSE);// 이벤트 초기화 [1/?]
```

먼저 설명드린 구조체를 초기화합니다. 그리고 여기서 해당 구조체의 필드에서는 **KEVENT**가 존재합니다. 이 구조체의 역할은 주로 스레드의 **실행을 중지**할 때 사용됩니다. <br>

그렇다면 "KEVENT 구조체는 무엇을 위해 사용되는가?"에 대한 물음이 있을 수 있습니다. <br>

이 콜백함수의 핸들러는 **최대한 빠르게 처리하고 return**되어야 하는 함수입니다. <br>

그러한 이유는, 프로세스가 실행되어 유저모드에서 보여지기 전까지, 이 핸들러 함수를 거치고 return이 되어야 다음 단계로 넘어가기 때문입니다. <br>

그러면 여기서 **의문**이 생길 겁니다. <br>

빠르게 리턴되어야 한다면서 왜 KEVENT로 멈추는건가? 이겠죠.<br>

이것은 반대로 빠르게 리턴하는 것을 고려하기 위해 잠시 있다가 사라지는 **포인터 구조체**의 데이터를 **다른 스레드**에서 복사하기 위함입니다. 

```c
HANDLE thread_handle;
PsCreateSystemThread(&thread_handle, THREAD_ALL_ACCESS, NULL, NULL, NULL, THREAD_for_send_to_process_data, &TMP_data_pack); // 프로세스 정보를 스레드에 전달
ZwClose(thread_handle);
/*
	호출한 스레드	<	THREAD_for_send_to_process_data	>	에서 해제해줄 때까지 무한정 대기
*/

KeWaitForSingleObject(&TMP_data_pack.event_for_saving, Executive, KernelMode, FALSE, NULL); // 이벤트 점유
```

위 코드는 실제로 콜백함수 핸들러에 있는 코드입니다. <br>

**PsCreateSystemThread** API를 이용하여 커널에서 **THREAD_for_send_to_process_data**를 호출하여 스레드를 생성하고, **TMP_data_pack**변수의 주소를 넘기고 있습니다. <br>

그리고 **KeWaitForSingleObject** API로 KEVENT를 점유하여 실행을 중단합니다. <br>

이제 핸들러는 **THREAD_for_send_to_process_data**스레드에서 종료할 때까지 무한히 점유되는 상태가 됩니다. <br>

```c
VOID THREAD_for_send_to_process_data(PPROCESS_ROUTINE_struct context) {
	/*
		데이터 이관
	*/
	PROCESS_ROUTINE_struct DATA = { 0 };
	DATA.is_process_creating = context->is_process_creating;
	DATA.eprocess_addr = context->eprocess_addr;
	DATA.process_pid = context->process_pid;
	DATA.EXE_bin = (PUCHAR)ExAllocatePoolWithTag(PagedPool, context->EXE_bin_size, 'FILE');
	memcpy(DATA.EXE_bin, context->EXE_bin, context->EXE_bin_size);
	DATA.EXE_bin_size = context->EXE_bin_size;
	DATA.sha256 = (PUCHAR)ExAllocatePoolWithTag(NonPagedPool, SHA256_String_Byte_Length - 1, 'SHA');

	memcpy(DATA.sha256, context->sha256, SHA256_String_Byte_Length - 1);  //블루스크린 발생 ( PID_to_EXE 실패해서 그걸 가져와버린듯) 

	DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, " PcreateProcessNotifyRoutineEx sha256_2 -> %s\n", DATA.sha256);
	ExFreePoolWithTag(context->sha256, 'SHA');
	/* 이벤트 점유 해제 */
	KeSetEvent(&context->event_for_saving, 0, FALSE); // 루틴함수는 이제 프로세스를 정상적으로 생성할 수 있게 된다. ( 단, 포인터는 이제 유효X ) 
```

이는 방금 생성된 **THREAD_for_send_to_process_data**의 초반부입니다. <br>

콜백함수 핸들러에서 얻은 **포인터 구조체**를 가져와서 따로 처리하기 위해 memcpy로 이전합니다.<br>

이전을 완료했다면, **KeSetEvent** API를 호출하여 **KeWaitForSingleObject** 로 점유하고 있던 KEVENT를 해제하도록 합니다. <br>

그럼 이제 지금부터는 편안하게 속도 상관없이 천천히 처리하여도 상관없습니다. 또한 프로세스가 여러번 생성될 때, 이 스레드도 똑같이 여러번 생성되어 처리됩니다. <br>

```c
PLIST A_node = CREATE_tag_LIST(sizeof(DATA.is_process_creating), (PUCHAR)&DATA.is_process_creating, NULL, &NODE_SIZE); // 1
```

위 코드에서 **PLIST** 구조체가 존재합니다. 이는 모니터링에 저장될 데이터를 **연결리스트**로 저장합니다. 그렇기 때문에 노드 하나하나가 각각의 데이터를 가집니다. <br>

PLIST?<br>

```c
typedef struct LIST {

	ULONG32 RAW_DATA_len;
	PUCHAR RAW_DATA; // 여기서 새로 동적할당하지 않고, 바로 전역변수에 축적된다. (결론= 전역변수에 저장 시, 비동기 처리 절대 불가능 ) 

	PUCHAR previous_addr;
	PUCHAR next_addr;
}LIST, * PLIST;
```

그렇다면 노드에 들어갈 데이터는 새로 동적할당되어서 저장되는가? 생각할 수 있습니다만, 이 스레드에서 연결리스트를 생성하고, 완전히 반환되기 까지는 매우 복잡한 로직이 포함되어 있으며, 추가적인 비동기 스레드를 생성하지 않습니다. 그렇기 때문에 PLIST 구조체에서 **RAW_DATA**필드 값에 저장할 때는, 그대로 변수의 주소만 붙여집니다. 이러한 의미는 **연결리스트는 임시적으로 생성되고 제거**됨을 이해해야합니다. <br>

### 연결리스트를 가지고 어떻게 쓰이고 제거되는가?<br>

그러면 연결리스트를 가지고 어떻게 쓰이고 제거되는가? 가 생각될 수 있습니다. <br>

```c
ULONG32 TYPE = (ULONG32)process_routine; // 타입명시
Send_or_Save(SAVE_RAW_DATA, &TYPE, &(PUCHAR)A_node, &NODE_SIZE, NONE);
```

위 코드는 중앙서버가 자동 파싱을 위해 저장하는 TYPE과, 모니터링된 다양한 정보를 취합하여 전역변수에 최종 저장하고 중앙서버에 전달하는 **Send_or_Save()**함수가 있습니다. <br>

이 함수의 과정을 설명드리면 다음과 같습니다.<br>

1. 연결리스트의 노드 마지막 주소, SAVE인지 SEND인지 확인하는 ENUM값 등을 받습니다.
2. SAVE과정일 때, 연결리스트를 순환하여 모든 노드의 **RAW_DATA**필드 값을 가져와 한 공간을 담는 변수를 동적할당하고 그안에 **길이-기반**으로 변환하고 전역변수에 APPEND합니다.(memcpy로 일일히 필드주소의 데이터를 복사합니다.)
3. SEND과정일 때, (2)과정에서 다양한 모니터링 TYPE이 저장된 전역변수의 데이터를 모두 중앙서버에게 전달하고, 초기화합니다. 

이러한 과정속에서도 하나씩 코드를 이해하려면 복잡할 수 있습니다. <br>

어쨌거나, 이 과정이 완료되면, 이 스레드에서 동적할당된 데이터를 모두 해제하고 완전히 반환합니다.
