---
layout: default
title: Hardware_ID
nav_order: 1
permalink: /pages/AGENT/Hardware_info
parent: AGENT
typora-root-url: ../../../
---

# **고유한 HardWare 정보를 얻는 방법을 설명합니다.**

### 쿼리를 통하여 SMBIOS를 추출하고, TYPE 1 과 TYPE 2를 추출합니다.



```c
// ACPI 테이블을 읽는 예제 코드 (매우 간단히 설명)
NTSTATUS status;
ULONG tableSize;
SYSTEM_FIRMWARE_TABLE_INFORMATION* firmwareTableInfo;
ULONG bufferSize = sizeof(SYSTEM_FIRMWARE_TABLE_INFORMATION) + 1024;

firmwareTableInfo = (SYSTEM_FIRMWARE_TABLE_INFORMATION*)ExAllocatePoolWithTag(PagedPool, bufferSize, 'RSMB'); //먼저 어느정도 길이를 가져와야하는 것으로 확인.(쿼리 전, Hint정보를 필드에 넣고 쿼리하기 때문) 
if (!firmwareTableInfo) {
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "메모리 공간 부족\n");
    return STATUS_INSUFFICIENT_RESOURCES;
}

RtlZeroMemory(firmwareTableInfo, bufferSize);
firmwareTableInfo->ProviderSignature = 'RSMB'; // ACPI 시그니처 설정
firmwareTableInfo->Action = SystemFirmwareTable_Get; // " Enum " 으로 펌웨어 정보 가져오도록함 



while (( status = ZwQuerySystemInformation(SystemFirmwareTableInformation, firmwareTableInfo, bufferSize, &tableSize) )  != STATUS_SUCCESS) {

// 선 할당 해제
ExFreePoolWithTag(firmwareTableInfo, 'RSMB');
firmwareTableInfo = NULL;

if (
    (status == STATUS_INVALID_INFO_CLASS) ||
    (status == STATUS_INVALID_DEVICE_REQUEST) ||
    (status == STATUS_NOT_IMPLEMENTED) ||
    (tableSize == 0)
    ) {
    //ExFreePoolWithTag(firmwareTableInfo, 'RSMB');
    return STATUS_UNSUCCESSFUL;
}
else if (status == STATUS_BUFFER_TOO_SMALL) {
    //ExFreePoolWithTag(firmwareTableInfo, 'RSMB');
    firmwareTableInfo = (SYSTEM_FIRMWARE_TABLE_INFORMATION*)ExAllocatePoolWithTag(PagedPool, tableSize, 'RSMB');
    if (!firmwareTableInfo) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "SystemFirmwareTableInformation 쿼리 결과 -> STATUS_BUFFER_TOO_SMALL 이기에, 재할당했지만 메모리 공간 부족;\n");
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlZeroMemory(firmwareTableInfo, tableSize);
    firmwareTableInfo->ProviderSignature = 'RSMB';
    firmwareTableInfo->Action = SystemFirmwareTable_Get;
    bufferSize = tableSize;  // bufferSize를 tableSize로 업데이트
}
else {
    //ExFreePoolWithTag(firmwareTableInfo, 'RSMB');
    return STATUS_UNSUCCESSFUL;
}

continue;

}


//쿼리 성공후 처리를 구현하라

DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "SystemFirmwareTableInformation 쿼리 성공!\n");



```
