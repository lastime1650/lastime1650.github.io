---
layout: default
title: GUI
nav_order: 6
permalink: /pages/GUI
has_children: true
typora-root-url: ../../
---



# **Python 의 TKinter로 구축한 에이전트 관리 program**

### 관리자가 자신의 LICENSE_ID를 기반으로 등록된 에이전트를 원격으로 모니터링된 정보를 감시하고, 명령을 내릴 수 있습니다. 

### <br>

TKinter모듈을 이용하여 에이전트를 원격으로 관리하는 어플리케이션을 구축하였습니다.<br>

먼저, [LICENSE_ID](https://lastime1650.github.io/pages/structure#%EC%97%90%EC%9D%B4%EC%A0%84%ED%8A%B8-agent) ,  GUI_ID를 포함하여 중앙서버에게 전달해야합니다. 이는 관리자를 식별하고, 이에 따라 관리자가 GUI 프로그램으로 에이전트를 관리하겠다는 의미를 중앙서버에서 식별토록 합니다. <br>

연결 된 후, GUI 프로그램은 중앙서버에게 LICENSE_ID를 기준으로 묶인 AGENT_ID를 가져오도록 요청합니다. 이는 [**길이-기반**](https://lastime1650.github.io/pages/structure#%EA%B8%B8%EC%9D%B4%EA%B8%B0%EB%B0%98-%EB%8D%B0%EC%9D%B4%ED%84%B0-%ED%8F%AC%EB%A7%B7---length-based--_-rawdata)으로 요청하는 형태입니다.<br>

이제 GUI프로그램은 자신의 LICENSE_ID를 통해 등록된 AGENT_ID들을 가져왔으니, 동적으로 버튼을 구성하도록 합니다. <br>

이 후,  **비동기 스레드**를 추가하여 가져온 모든 AGENT_ID에 대하여 살아있는 지 주기적으로 중앙서버에게 여부를 요청합니다. <br>

에이전트가 Offline 상태일지라도, DB에는 AGENT_ID를 기반으로 에이전트 정보가 저장되어 있으므로, 버튼과 상호작용시 필요한 정보를 얻을 수 있습니다. <br>

에이전트에게 명령을 내려야 할 때에도 마찬가지입니다만, Offline상태 일지라도, 요청을 하면 중앙서버는 DB에 명령 정보(대표적으로 **보호조치 Table**)를 저장하도록 합니다. 만약, 에이전트가 Online상태인 경우 바로 중앙서버는 에이전트에게 직접적으로 명령을 내릴 수 있겠지만, Offline상태인 경우일 때에는 Online최초 상태가 되었을 때 미리 구현한 Initialize()함수를 호출하여 에이전트 커널 드라이버가 본격적인 활동을 시작하기 전에 필요한 정보를 Load과정에서 적용하도록 구현하였습니다. 

