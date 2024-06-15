---
layout: default
title: Agent_Utility
nav_order: 6
permalink: /pages/AGENT/AGENT_Util
parent: AGENT
typora-root-url: ../../../
---

# **커널에서 구현한 유틸리티 함수들을 설명합니다.**

### 이 유틸리티 함수는 내부 로직을 처리하는데, 가장 중요한 역할을 해도 과언이 아닙니다.

<br>

다음과 같은 유틸리티를 개발하였습니다. 



- 초단위를 인수로 넘기면 딜레이가 걸리는 함수
- 소켓을 구축하는 함수
- UNICODE_STRING 을 ANSI_STRING으로 변환하는 함수 ( 역변환도 가능 )
- ZwQuerySystemInformation() 쿼리 활용 하여 하드웨어 정보 ( SMBIOS ) 추출
- ZwQueryInformationProcess() 쿼리 활용 통한 프로세스 Handle을 통하여 프로그램 절대경로 추출
- HANDLE, 과 절대경로를 건네면 파일 입출력 (읽기, 쓰기)이 가능한 함수
- bcrypt.h 를 이용하여 바이트값을 전달하면 SHA256으로 해싱하는 함수
- PID 에서 EPROCESS를 이용하여 HANDLE값 추출

