---
layout: default
title: AGENT_Socket
nav_order: 1
permalink: /pages/AGENT/Socket
parent: AGENT
has_children: true
typora-root-url: ../../../
---

# **커널에서 중앙서버간 Socket통신을 설명합니다.**

### Windows Kernel Driver의 Winsock.h(wsk.h)으로 개발되어 수집된 데이터를 중앙서버에게 정해진 포맷으로 전달합니다.

<br>

여기서는 어떻게 Windows Kernel Driver에서 Socket을 구축하고 중앙서버와 어떻게 TCP - Client 통신 요청하는 지 알아봅니다. <br>



<br>

다양한 정보는 하위 디렉터리에서 제공됩니다.

-----

## Winsock + Kernel 을 이용한 TCP 클라이언트

Winsock에서도 커널을 사용할 수 있도록 Microsoft에서 모듈을 구현하였습니다.  <br>
또한 이 코드의 레퍼런스는 다음 [Github](https://github.com/wbenny/KSOCKET?tab=readme-ov-file) 주소를 사용하였습니다.



실제 API를 사용할 때는 줄여서 wsk으로 표현되어 사용됩니다. <br>

먼저 WSK의 개체는 총 2가지로 구성되어 있습니다. <br>

1. WSK_CLIENT
2. WSK_SOCKET 

```c
typedef struct _WSK_CLIENT_NPI {
    PVOID                        ClientContext;
    CONST WSK_CLIENT_DISPATCH   *Dispatch;
} WSK_CLIENT_NPI, *PWSK_CLIENT_NPI;
```

여기서는 에이전트가 중앙서버에 TCP 통신하는 입장이므로, WSK_CLIENT 개체로 선택되었습니다. <br>

이후 WskSocket API를 통해 **SOCK_STREAM** 소켓 객체를 생성하여 중앙서버에게 TCP 요청을 하는 방법을 간략히 설명합니다.<br><br>

```c
WSK_CLIENT_DISPATCH  WskDispatch = { MAKE_WSK_VERSION(1,0), 0, NULL };

WSK_CLIENT_NPI WskClientNpi;
WskClientNpi.ClientContext = NULL;
WskClientNpi.Dispatch = &WskDispatch;
```

WSK_CLIENT_NPI에서 디스패치 (기본정보) 값을 초기화합니다. <br>

```
WSK_REGISTRATION     WskRegistration = { 0, };

if (!NT_SUCCESS(WskRegister(&WskClientNpi, &WskRegistration))) {
    return STATUS_UNSUCCESSFUL;
}
DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "WskRegister 성공");

if (!NT_SUCCESS(WskCaptureProviderNPI(&WskRegistration, WSK_INFINITE_WAIT, &WskProvider))) {
    return STATUS_UNSUCCESSFUL;
}
```

그런 다음 순차적으로 wskRegister API로 개체를 등록하고, 이에 대한 공급자(Provider)를 포인터로 얻기 위하여 WskCaptureProviderNPI API를 사용합니다.<br>

```
typedef struct _KSOCKET_ASYNC_CONTEXT
{
    KEVENT CompletionEvent;
    BOOLEAN is_Locked_by_always_receive;
    PIRP Irp;
} KSOCKET_ASYNC_CONTEXT, * PKSOCKET_ASYNC_CONTEXT;

typedef struct _KSOCKET
{
    KMUTEX Mutex;

    PWSK_SOCKET	WskSocket;

    union
    {
        PVOID WskDispatch;

        PWSK_PROVIDER_CONNECTION_DISPATCH WskConnectionDispatch;
        PWSK_PROVIDER_LISTEN_DISPATCH WskListenDispatch;
        PWSK_PROVIDER_DATAGRAM_DISPATCH WskDatagramDispatch;
        PWSK_PROVIDER_STREAM_DISPATCH WskStreamDispatch;

    };

    KSOCKET_ASYNC_CONTEXT AsyncContext;
} KSOCKET, * PKSOCKET;
```

여기 꽤나 복잡한 구조체가 존재합니다. KSOCKET 구조체는 임의의 구조체입니다.  <br>

wsk으로 통신하면 IRP 요청으로 패킷의 수신이나 송신 상태를 무조건 받아 처리해야하는 **KSOCKET_ASYNC_CONTEXT**  구조체와 소켓을 구성하는 WSK_SOCKET, Dispatch, 상호배제를 위한 뮤텍스가 필드에 존재하고 있으며, 이는 송수신 과정에서 필요한 데이터들을 담았다고 생각하면 됩니다.<br>

```c
/*초기화*/
KeInitializeEvent(&(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent), SynchronizationEvent, FALSE); // 이벤트 초기화

KeInitializeMutex(&(((PKSOCKET)(*NewSocket_parm))->Mutex), 0);

((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp = IoAllocateIrp(1, FALSE); // IRP초기화 
if (((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp == NULL) {
    ExFreePoolWithTag(((PKSOCKET)(*NewSocket_parm)), 'ABCD');
    return STATUS_UNSUCCESSFUL;
}

IoSetCompletionRoutine( // IRP 콜백루틴 적용
    ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp,
    &KspAsyncContextCompletionRoutine,
    &(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent),
    TRUE,
    TRUE,
    TRUE
);
```

초기화 코드 부분입니다. <br>

Socket으로 통신할 때, 정상적으로 송신되었는 지, 또는 수신되었는 지까지의 대기시간을 구현하기 위하여 **KEVENT**가 사용됩니다. 이를 이용하면, WskSend(TCP전송) 및 WskReceive(TCP수신) 등 Wsk상호작용 API를 호출하고, **IoSetCompletionRoutine** API에 등록한 IRP 콜백함수로부터 Socket송수신의 응답을 받고(동기적) 소켓을 처리하도록 구현해야합니다. 만약. 이러한 동기처리를 구현하지 않으면, 순서가 틀어지게 되며, BSOD가 발생될 수 있습니다. IRP와 SOCKET,, 이들은 어떻게 사용되는 지 바로 아래에서 설명드립니다.  <br>

{: .note }

특히 초기화 과정에서 IRP가 왜 사용되고, 이 IRP가 SOCKET 통신과 어떤 흐름을 가지는 지 가 가장 중요합니다. 

<br>

```c



IoReuseIrp(((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp, STATUS_UNSUCCESSFUL);

IoSetCompletionRoutine( // IRP 콜백루틴 적용
    ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp,
    &KspAsyncContextCompletionRoutine,
    &(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent),
    TRUE,
    TRUE,
    TRUE
);

/*본격 소켓 제작*/
NTSTATUS status = WskProvider.Dispatch->WskSocket(
    WskProvider.Client,
    AF_INET,
    SOCK_STREAM,
    IPPROTO_TCP,
    WSK_FLAG_CONNECTION_SOCKET,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp
);
/*이벤트 대기확인*/
if (status == STATUS_PENDING) {
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "STATUS_PENDING ...");
    KeWaitForSingleObject(
        &(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent),
        Executive,
        KernelMode,
        FALSE,
        NULL
    );

    status = ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp->IoStatus.Status;
}
/*소켓 제작 마무리*/
if (NT_SUCCESS(status))
{
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "소켓제작성공!");
    ((PKSOCKET)(*NewSocket_parm))->WskSocket = (PWSK_SOCKET)((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp->IoStatus.Information;
    ((PKSOCKET)(*NewSocket_parm))->WskDispatch = (PVOID)((PKSOCKET)(*NewSocket_parm))->WskSocket->Dispatch;

}
```

위 코드는 WskSocket API를 이용하여 WSK_SOCKET 구조체 포인터를 획득하는 과정입니다. 실제로 WskProvider.Dispatch포인터의 필드에서 존재하는 Wsk.. API들은 요청하면 항상 IRP 콜백함수로부터 동기적으로 응답을 기다려야합니다. 이는 이제 아시다시피 비동기 처리가 될 수 없습니다. <br>

코드를 확인하면 WskSocket API를 호출하기 전에, IRP를 초기화하고, IRP의 콜백함수를 지정합니다. 이는 동기처리를 위한 준비단계이므로, Wsk관련 API를 사용하기 전에 **무조건** 실행되어야 하는 로직입니다. <br>

WskSocket API를 호출하는 단계에서 이에 대한 반환 값이 STATUS_PENDING(대기하라)이라는 NTSTATUS CODE가 반환되는 경우가 있습니다. 이 경우를 대비하여 KeWaitForSingleObject API로 초기화된 KEVENT를 점유하여 스레드를 중단하고, IRP의 콜백함수 호출로 인하여 KeSetEvent() API가 호출되어 점유를 해제하도록 하여 스레드를 재개하도록 해야합니다. 이어서, 이렇게 비동기적으로 처리하도록 하고, 이후 Wsk API가 성공적으로 처리되었는 지에 대한 여부는 IoStatus.Status의 NTSTATUS CODE값을 확인하여 최종 처리하면 됩니다. <br>

{: .note }

IRP초기화 -> IRP콜백함수 정의 -> WSK API호출 -> PENDING 응답인 경우 KEVENT로 스레드 중단 -> IRP콜백함수에서 KeSetEvent() 호출할 때까지 대기 -> 스레드 재개될 때, IoStatus.Status값으로 최종 결과 확인

<br>

```c
IoReuseIrp(((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp, STATUS_UNSUCCESSFUL);

IoSetCompletionRoutine(
    ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp,
    &KspAsyncContextCompletionRoutine,
    &(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent),
    TRUE,
    TRUE,
    TRUE
);

/*TCP 보내기 전 세션 중 서버로부터 받기 준비 세팅 */
SOCKADDR_IN LocalAddress;
LocalAddress.sin_family = AF_INET;
LocalAddress.sin_addr.s_addr = INADDR_ANY;
LocalAddress.sin_port = 0;

status = ((PKSOCKET)(*NewSocket_parm))->WskConnectionDispatch->WskBind(
    ((PKSOCKET)(*NewSocket_parm))->WskSocket,          // Socket
    (PSOCKADDR)&LocalAddress,   // LocalAddress
    0,                          // Flags (reserved)
    ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp    // Irp
);
/*이벤트 대기 해야하는지 확인*/
if (status == STATUS_PENDING) {
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "STATUS_PENDING ...");
    KeWaitForSingleObject(
        &(((PKSOCKET)(*NewSocket_parm))->AsyncContext.CompletionEvent),
        Executive,
        KernelMode,
        FALSE,
        NULL
    );

    status = ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp->IoStatus.Status;
}
/*소켓 제작 마무리*/
if (!NT_SUCCESS(status))
{
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "바인딩 실패!");
    return STATUS_UNSUCCESSFUL;
}
```

다음은 Bind을 구현합니다. 이는 UDP와 다르게 TCP는 신뢰성이므로, 서버의 ACK도 받아서 처리되어야 합니다. <br>

단, 에이전트는 절대 TCP 서버 Socket이 되는 것이 아닙니다. Socket에서의 IP Address를 정의할 때 사용하는 SOCKADDR_IN 구조체에서 port를 0으로 하고 있으며, **LocalAddress.sin_port = 0;** . LocalAddress.sin_addr.s_addr 의 값을 **ANY**로 구성하여 Bind하는 것을 확인할 수 있습니다. <br>

```c
// 재사용 초기화 생략...

SOCKADDR_IN RemoteAddress;

 RemoteAddress.sin_addr.S_un.S_un_b.s_b1 = (UCHAR)ip_byte[0]; //A
 RemoteAddress.sin_addr.S_un.S_un_b.s_b2 = (UCHAR)ip_byte[1]; //B
 RemoteAddress.sin_addr.S_un.S_un_b.s_b3 = (UCHAR)ip_byte[2]; //C
 RemoteAddress.sin_addr.S_un.S_un_b.s_b4 = (UCHAR)ip_byte[3]; //D

 RemoteAddress.sin_family = AF_INET; // IPv4
 RemoteAddress.sin_port = RtlUshortByteSwap(Server_port); // Port 번호ㅇ

 status = ((PKSOCKET)(*NewSocket_parm))->WskConnectionDispatch->WskConnect(
     ((PKSOCKET)(*NewSocket_parm))->WskSocket,          // Socket
     (PSOCKADDR)&RemoteAddress,     // RemoteAddress
     0,                             // Flags (reserved)
     ((PKSOCKET)(*NewSocket_parm))->AsyncContext.Irp    // Irp
 );
 
 //  생략..
```

 

이번에는 WskConnect API를 이용하여 최초 중앙서버에 TCP요청을 시도하는 코드입니다. SOCKADDR_IN 구조체에 중앙서버의 IP와 PORT를 알맞게 저장하고 API 인자에 집어넣습니다. <br>



이 과정까지 완료되어 최종적으로 **STATUS_SUCCESS** 를 반환받았다면 지금부터 중앙서버와 TCP 세션을 통하여 데이터를 송수신할 수 있습니다. WskSend, WskReceive를 잘 활용하고, 데이터를 전달하거나 받을 때는 

```c
 WSK_BUF WskBuffer;
 WskBuffer.Offset = 0;
 WskBuffer.Length = Buffer_size;
 WskBuffer.Mdl = IoAllocateMdl(Buffer, (ULONG)WskBuffer.Length, FALSE, FALSE, NULL);

// 해제는 IoFreeMdl()
```

WSK_BUF 구조체를 활용하고, IoAllocateMdl API를 이용하여 버퍼를 구성하여 올바르게 데이터를 전송하거나 받도록 하면 됩니다. 
