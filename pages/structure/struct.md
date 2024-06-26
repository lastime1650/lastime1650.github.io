---
layout: default
title: Struct
nav_order: 2
permalink: /pages/structure
typora-root-url: ../../
---

# **보안 솔루션의 구성도를 설명합니다.**

### 현재 적용된 보안 솔루션의 구조를 사진과 함께 서술하여, 전체적인 흐름을 서술합니다.

![img](/img/struct_img.png)



이들은 모두 서로 떨어진 환경에서도 작동되도록 구현되었으며 통신을 위해 Socket을 이용합니다.



{: .highlight }

Socket으로 주고 받는 RAW_DATA포맷은 상황에 따라 다르지만 주로  **"길이-기반"구조**로 전달됩니다. 

---



## 중앙서버 (RUST)

<br>

RUST언어로 개발된 중앙서버는 모든 노드의 요청을 처리합니다. 

<br>

각 요청하는 노드는 RUST와 **Socket**통신으로 이루어지게 되며, 노드에 따른 처리를 개별적으로 하기 위하여 **병렬 스레드**로 구현됩니다. 



{: .note }

병렬스레드에서 충돌을 방지하기 위하여 **Arc< Mutex< T > >** 타입으로 인스턴스들을 관리합니다. 



---



## 에이전트 (AGENT)

<br>

C 언어로 개발된 에이전트는 현재 **Windows Kernel Driver**로 작성되어, Windows 운영체제만 지원됩니다. 





{: .warning }

> 커널로 제작된 에이전트(driver)이므로 메모리 관리에 신경을 써야합니다.



<br>

또한 중앙서버(RUST)와 Socket통신 하므로 원격으로 자신이 수집한 시스템 정보를 전달합니다.



{: .note-title }

>커널 에이전트는 무슨 정보를 중앙서버에게 보내는가?
>
>
>
>시스템 정보는 주로 콜백루틴, 하드웨어 정보등이 포함됩니다.



<br>

중앙서버가 자신(Host)을 식별하기 위해 **AGENT_ID**와 **LICENSE_ID ** 그리고 **HARDWARE**정보를 전달합니다.



{: .important-title }

>AGENT_ID ?
>
>
>
>**AGENT_ID**는 호스트를 식별하는 SHA-512값입니다. 이 정보는 중앙서버에서 발급하며, 최초 에이전트 통신시 발급한 AGENT_ID를 사용하여 에이전트를 식별할 수 있습니다.



{: .important-title }

>LICENSE_ID?
>
>
>
>**LICENSE_ID**는 보안 솔루션의 사용자 계정에 대한 요금제 일련번호를 의미합니다. 이는 라이선스 ID마다 중앙서버로부터 솔루션을 제공받을 수 있는 에이전트 수가 **한정**되어 있습니다. 



{: .important-title }

>HARDWARE 정보?
>
>
>
>**HARDWARE 정보**는 에이전트에서 중앙서버에게 연결접속 전에 **SMBIOS**정보를 얻어 **Type1(시스템)** 과 **Type2(마더보드)** 를 추출한 정보를 중앙서버에게 전달합니다. 이 후, 중앙서버는 이 정보를 취합하여 **HARDWARE_ID**를 생산하며 이는 SHA-512로 해싱된 값입니다.  
>
>
>
>HARDWARE_ID를 도입한 이유는 호스트가 Offline 상태에서도 중앙서버에서도 기억하여 언제든지 호스트가 참여했을 때 기억을 유지할 수 있도록 돕습니다.



---





## 에이전트 관리자 프로그램 GUI - TKinter

<br>

Python의 TKinter로 테스트 구축한 **에이전트 관리 프로그램**입니다. 

<br>

이 프로그램은 에이전트가 수집한 데이터를 Chart로 정리해서 볼 수 있거나, 직접 AGENT_ID를 통하여 식별된 에이전트에 대하여 명령을 내릴 수 있습니다.



{: .note }

에이전트 관리 프로그램에서 에이전트의 수집된 정보를 보기 위해서 **중앙서버(RUST)**에 요청하여 얻어야합니다. 



---







## AI 서버 

<br>

Python의 Tensorflow 모듈을 이용하여 모델을 구축한 서버입니다.

<br>

이 서버는 단순히 판단을 하는 서버가 아닌 자동으로 Malware Bazaar API를 사용하여 맬웨어를 수집하고 자신의 서버에 저장할 수 있습니다.  <br>

또한 맬웨어 탐지의 정확성을 위하여 AI 서버에게 AI판단을 요청하는 중앙서버(RUST)에서는 자신의 VirusTotal API값을 포함해서 전달해야만 합니다. 



{: .highlight }

**VirusTotal**은 훈련,예측(판단)용! , **Malware_Bazaar**는 맬웨어를 자동수집하여 데이터세트에 기여합니다. 



{: .note }

구현된  모델은 **DNN**입니다. 이미지 방식 **CNN**의 경우 개발중인 노트북 환경의 부족함에 의해 진행하지 않았습니다. 



-----



## 데이터베이스 ( Mysql - Mariadb )

<br>

Mysql 쿼리가 가능한 MariaDB로 구축하였습니다.

<br>

이 데이터베이스는 주로 모든 에이전트에 대한 정보를 수집 및 로깅합니다. <br>



그뿐만 아니라,  솔루션을 제공하기 전에 DB에 저장된 데이터를 쿼리하여 검증하거나 결과를 저장하는데에도 사용됩니다. 



{: .warning }

> 해당 데이터베이스는 에이전트, 사용자정보, 라이선스정보 등등이 포함되므로 엄격하게 관리되어야 합니다.



----

## 길이기반 데이터 포맷  ( Length-Based  _ RawData)

## <br>

이 포맷은 **네트워크 간 버퍼 공유** 인 Socket통신에서 **프로그래밍 언어를 불문**하고 모든 언어에서 해석될 수 있는 데이터 형식입니다.<br>

이는 프로토콜을 정의한 것이 아니며, RAW데이터 단에서 유의미한 데이터를 **길이라는 개념**을 도입하여 항상 동적의 길이를 가진 데이터를 **자동화 파싱**을 위해 설계한 것입니다.<br>

더 나아가서 **확장된 길이기반 형식**이 있습니다. 이는 일반적인 길이기반 형식과 다르게, **다양한 의미를 가지는 길이기반 형식**으로, 쉽게 이해하자면, 일반적인 길이기반 포맷이 **여러개**존재한다라고 보면 됩니다. <br><br>



이 형식의 기본 구조는 2가지로 나뉩니다. 

1. 동적길이를 정의하는 고정된 길이 값과 동적길이의 데이터
2. **고정된 명령헤더**와 동적길이를 정의하는 고정된 길이 값과 동적길이의 데이터

<br>

(1)의 구조는 이론을 설명하기 위한 단순한 형태입니다. <br>

처음에는 **동적길이를 정의하는 고정된 길이 값**이라는 용어가 존재하는데, 이는 실제로 전송할 동적데이터의 "길이"를 의미하는 것입니다. <br>

그렇기 때문에 여기서 구현할 때는 4Byte로 정의하여 4Byte내에서 표현가능한 수만큼 동적데이터를 전달할 수 있는 것이 됩니다. <br>

또한 그 다음 주소부터는 무조건 그 길이만큼 동적 데이터가 위치해야합니다. <br>

그러나, 길이 전체 이해하지 못하는 C언어가 있습니다. 이는 따로 길이를 알려줘야하는데, 소켓버퍼를 받을 때, 버퍼의 끝을 따로 알려줘야하는 문제가 있습니다. 그렇기 때문에, 맨 뒷 부분에 여기서는 **동적길이를 정의하는 고정된 길이의 크기Byte**길이 만큼의 Tail을 붙여줘야합니다. <br>

예를 들어 저는 4Byte로 고정되게 하였고, 최대 표현가능한 수 만큼 동적 데이터를 전달할 수 있습니다.(동적데이터 개/당)그리고 이를 무한히 반복할 때, 멈추기 위해서 **4Byte길이의 임의의 Tail Byte**값을 맨뒤에 저장합니다. 이 Byte를 디코딩하면 "_END" 라는 문자열이 생깁니다. <br>

{: .note }

>파싱된 데이터를 관리하기 위해서는 각 언어에 따라서 알아서 저장해야합니다. <br>
>
>파이썬의 경우는 list, <br>
>
>RUST의 경우에는 Vec< u8 > <br>
>
>C의 경우에는 연결리스트가 될 수 있습니다.
>
>



<br><br>

여기서 왜 **고정된 길이를 나타내는 크기 Byte**와  **맨 뒷부분의 Byte**가 일치해야하냐는 점입니다. <br>

이러한 이유는 "자동화 파싱"때문입니다. 4byte가 자리하는 부분은 동적데이터의 길이를 나타내는 Byte값뿐만 아닌, 이것이 끝인지 판단하는데에 사용되기 때문입니다.<br>

```python
    def Parsing_for_loop(self, DATA:bytes, start_index:int = 4, last_index:int = 8)->list: #파싱 과정중에서 반복적 작업이 필요한 경우가 있음
        Return_bytes_list = []
        while True:
            if DATA[start_index:last_index] == "_END".encode()[start_index:last_index]:
                return Return_bytes_list
            else:
                #print(DATA[start_index:last_index])
                RAW_DATA_len:int = self.byte_to_int( DATA[start_index:last_index] ) # Raw_data 길이 추출
                #print(RAW_DATA_len)

                start_index = last_index
                last_index = last_index + RAW_DATA_len
                RAW_DATA:bytes =  DATA[start_index:last_index] # Raw_data 가져옴
                Return_bytes_list.append( RAW_DATA )


                start_index = last_index
                last_index = last_index + 4 # 다음을 위해 4바이트 이동
```

위 코드는 파이썬의 AI_Server에 구현된 **길이기반 형식 자동화 파싱 메서드**입니다. <br><br>

(2)는 **고정된 명령헤더**라고 칭하였습니다. 이는 (1)에서 정의한 **고정된 길이 값의 Byte길이**와 같은 Byte길이로 정의할 필요가 없습니다. 어차피 이는 맨 앞의 Header를 의미하고, **Header다음부터는 길이 Byte**가 위치하기 때문입니다. <br>

이 명령헤더는 **길이기반의 형식의 모든 데이터가 무엇을 의미하는 지 해석**하는데 사용됩니다.<br>

<br>

<br>

이를 체험할 수 있는 레포지토리를 만들었습니다.<br>

[체험하기](https://github.com/lastime1650/Length_Based_Dynamic_Socket_Buffer){: .btn }
