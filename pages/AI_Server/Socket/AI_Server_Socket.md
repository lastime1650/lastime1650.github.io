---
layout: default
title: AI_Server_Socket
nav_order: 1
permalink: /pages/AI_Server/Socket
parent: AI_Server
typora-root-url: ../../../
---



# **AI서버와 중앙서버간 통신 과정을 설명합니다.**

### 중앙서버는 AI서버에게 자신의 VirusTotal API, EXE binary, SHA256등을 전달하여 AI요청을 하고 판단결과인 y값, float형태의 byte를 넘깁니다.<br>

<br>

---

## 풀이

<br>

이 AI서버와 중앙서버의 관계는 1:N 관계입니다. <br>

즉, AI서버는 Cloud형태로 운영될 수 있으며, 중앙서버 자체를 구매하여 **온프레미스**환경(단, DB와 WEB서버는 제외)을 구축한 고객을 위하여 제공하는 AI 서비스입니다. <br>

AI서버는 100% Python으로 구축되었습니다. 또한 **Threading** 모듈을 이용하여 Client에서 요청이 올때 비동기 스레드를 생성하여 별도의 처리를 수행하고, **모델간 충돌**을 방지하기 위해 모델을 총 관리하는 클래스의 인스턴스를 자원공유하는 형태로 구현됩니다. <br>

또한 맬웨어 데이터 수집의 경우 자동화로 구현하였기 때문에, **비동기 프로그래밍이 중요**합니다.<br>

데이터세트는 DISK에 저장된 CSV포맷 파일의 형태로 총 2개가 저장됩니다. <br>

1. 악성인지 아닌지 판단하는 데이터세트 (단, 파일경로, 파일SHA256, y판단값 3개의 칼럼만 존재)
2. 중앙서버들이 요청하여 얻은 VirusTotal 의 결과를 담은 데이터세트 

특히 (2)에서 경제적 이득을 크게 얻습니다. <br>

VirusTotal의 API를 이용하여 분석요청하는 횟수는 정해져 있으며, 이를 크게 리미트가 적용된 라이선스를 판매하는 경우 몇천만원 후반의 가격이 발생합니다. 이러한 문제를 해결하기 위한 방법이 있습니다.<br>

AI서버에서 하나의 API를 가지고 훈련 및 예측시에 사용하지 않고, VirusTotal의 API부담은 Client가 부담하도록 AI요청시 전달받도록 구현하는 것입니다. 

---

## 기술적인 설명

<br>

```python
class AI_server():
    '''
        Server와 데이터를 송수신 할 때 사용되는 command
    '''
    Static_analyse = 1 # 정적분석요청 Server -> AI
    Dynamic_analyse = 2 # 동적분석요청 Server -> AI
    Vt_analyse = 3 # VirusTotal 분석요청 Server -> AI

    Static_analyse_with_Vt = 4 # 정적분석 + Vt 분석
    Dynamic_analyse_with_Vt = 5  # 동적분석 + Vt 분석

    ALL_analyse = 6 # 순서대로 정적->동적->Vt 를 한꺼번에 수행한다.

    Yes = 1029  # 공용 ( 대부분은 AI의 응답용 )
    No = 1030  # 공용 ( 대부분은 AI의 응답용 )
    '''
    '''

    Server_Sock = None # 소켓
```

위 코드는 중앙서버의 요청을 Receive하고 AI처리의 결과를 반환하는 클래스입니다.<br>

또한 클래스내에서 사용되는 변수들이 존재합니다. 이들은 중앙서버간의 소켓통신에서 데이터의 의미를 이해하고자 하는 명령을 정의한 것입니다.<br><br>



```python
    def __init__(self, Server_IP :str, Server_PORT:int):
        self.Server_Sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.Server_Sock.bind((Server_IP,Server_PORT))

        #self.Share_Variable_for_AI_MODELS = AI_Models_Manager_Class("SHARE_ai_exe_dataset.csv")
        self.AI_instance = AI_Instance(
            # Vt_API="b081a247f8bde7d98337dec2a44b9dff7d1eba19bd99a20c04395cea831177ee",
            Server_VT_api='b081a247f8bde7d98337dec2a44b9dff7d1eba19bd99a20c04395cea831177ee',
            VT_Csv_Path='C:\\Users\\Administrator\\PycharmProjects\\pythonProject\\AI_Server\\static_analyse\\Vt\\VT_analysed.csv',
            Main_Csv_Path='C:\\Users\\Administrator\\PycharmProjects\\pythonProject\\AI_Server\\static_analyse\\Vt\\SHARE_ai_exe_dataset.csv',
            Save_Index='C:\\Users\\Administrator\\PycharmProjects\\pythonProject\\AI_Server\\static_analyse\\Vt\\idk2'

        )
```

다음은 생성자 부분입니다.<br>

여기서는 Server 소켓을 구성하고, 단 하나의 AI모델을 관리하는 인스턴스를 생성합니다. 그리고 **이 인스턴스를 다양한 Client에서 자원을 점유**해서 사용하는 형태입니다. ( Threading.Lock() )<br><br>



```python
    def Start_Server(self):
        if( self.Server_Sock == None ):
            print("먼저 서버 소켓을 구축하십시오")
            return

        self.Server_Sock.listen(9999999)

        while True:
            print("기다리는중")
            Client_Sock, Client_INFO = self.Server_Sock.accept()
            threading.Thread(
                target=self.Client_Process, 
                args=(
                    Client_Sock, 
                    Client_INFO,
                    self.AI_instance #중요
                )
            ).start()
```

다음은 AI서버가 중앙서버의 AI 요청을 지속적으로 받고, 독립적인 처리를 위해 스레드를 생성하는 부분입니다.<br>

이 메서드를 실행하기 위해서는 소켓 인스턴스가 유효하여야만 합니다. 그리고 **threading.Thread()**를 통하여 스레드를 생성할 때, 소켓의 **accept()** 메서드의 결과(중앙서버 클라이언트 소켓 객체)를 적절히 넘겨 결과를 반환하는데 사용하도록 합니다. <br><br>

```python
 def Client_Process(self, Client_Sock:socket, Client_INFO, AI_inst_parm:AI_Server.static_analyse.Vt.NEW_AI_instance.AI_Instance):
        Client_INFO[0]# 클라이언트 IP
        Client_INFO[1]# 클라이언트 PORT
        print(f"{Client_INFO[0]}:{Client_INFO[1]} 님 환영합니다!! AI_inst_parm주소->{print(AI_inst_parm)}")
        while True:
            '''
                서버의 요청을 무한 리시브 한다
                {ENUM 4byte (le) } + {length + Raw_DATA} + {_END}
            '''

            Server_Data = self.RECEIVE_from_Server(Client_Sock) # 서버로부터 데이터 받기


            Parsing_Result = self.Parsing(Server_Data, AI_inst_parm) # 파싱 진행! ( Parsing -> Analyser )
            print(f"Parsing_Result->{Parsing_Result}")
            if Parsing_Result == None:
                # 실패!

                self.SEND_to_Server(Client_Sock, bytes( self.int_to_4byte( self.No ) )) # 실패전송

                print("[AI] 실패 전송 완료")
                continue
            else:
                ############### 성공!
                Ready_data:bytes = bytes( self.int_to_4byte(self.Yes) + Parsing_Result  ) # 데이터 마지막 "_END"는 Parsing 각 분석에서 만들어짐

                self.SEND_to_Server(Client_Sock, Ready_data)  # 최종 분석 결과 전송

                print("[AI] 성공 전송 완료")
```

다음은 비동기 스레드 부분입니다. 여기서는 중앙서버와 AI서버간 1:1 관계를 의미하는 공간입니다.<br>

여기 안에서는 실제 중앙서버가 AI요청을 위해 필요한 EXE바이너리값, SHA256, VirusTotal API 문자열을 제공받도록 합니다.<br>

중요한 점은 이 제공받은 데이터의 형식은 **"길이-기반"**형식으로 이루어져 있습니다. 그렇기 때문에 파이썬 내부 로직에서 파싱할 때,
반복문으로 **index**를 기준으로 Bytes데이터를 **slice**하여 필요한 데이터를 **struct**모듈을 이용하여 숫자로 변환하거나 그대로 사용(Binary)하여 AI를 처리하도록 합니다. 