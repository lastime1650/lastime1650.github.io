---
layout: default
title: Hardware_ID
nav_order: 2
permalink: /pages/AGENT/Hardware_info
parent: AGENT
typora-root-url: ../../../
---

# **고유한 HardWare 정보를 얻는 방법을 설명합니다.**

### 쿼리를 통하여 SMBIOS를 추출하고, TYPE 1 과 TYPE 2를 추출합니다.



------



HardWare 정보는 Kernel Driver가 메모리에 Load된 호스트가 재부팅되어도, 원격지의 중앙서버(RUST)가 AGENT_ID가 정확한 호스트를 언제든지 가르키도록 하기 위한 **식별정보**입니다. <br>

그렇기 때문에 이는 커널 내부 로직에서 SMBIOS를 쿼리하여 추출하고, 얻은 동적 버퍼 만큼 반복하여 정확하게 SMBIOS를 타입을 식별하여 가져오는 것이 중요합니다. <br>

또한 SMBIOS를 가져왔다 한들, SMBIOS의 TYPE이 의미하는 것을 인지하고 있어야합니다. 여기서는 절대 변함이 없는 Hardware정보만을 수집하는 것이 목표로 두었기 때문에, 이는 TYPE-1 과 TYPE-2이 이에 해당될 수 있습니다. <br>

수집 된 정보는 커널의 **전역변수**인 **Driver_ID**변수에 저장되어, 중앙서버(RUST)와 최초 TCP세션을 맺을 때 초기에 전송하는 데이터입니다. 



------

