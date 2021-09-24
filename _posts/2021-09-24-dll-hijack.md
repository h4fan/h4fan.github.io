---
layout: post
title:  dll hijack 学习
tags: [windows系统编程]
---

# 声明
本文所涉及程序均在本地运行测试，用于学习目的，请勿用于非法用途。

# 前言
学习一下dll hijack。 
dll proxy 可以根据链接1进行实验，本文给一个dll hijack的例子。

# 程序分析
程序见文末。

参见链接2，dll被加载时，DLL_PROCESS_ATTACH分支会被调用，所以可以在里面写对应的逻辑完成测试，本程序以一个MessageBox弹窗进行展示。


# 运行
运行processmonitor，设置filter，`Path` end with `.dll`,`Result` contains `NOT FOUND`,`Process Name`contains`filezilla`(换成你要分析的程序)。  
运行filezilla，可以看到缺少dll，可以将生成的dll改名为对应缺少的dll名称，放到filezilla目录下面，重新运行filezilla，发现出现我们的dll弹窗。成功。（有的名称可以成功，有的不能成功，多试几个名称。彩蛋留给读者去测试。）


# 源代码
```cpp
// dllmain.cpp : Defines the entry point for the DLL application.
#include "pch.h"
#include <stdio.h>
#include <stdlib.h>

DWORD WINAPI DoMagic2(LPVOID lpParameter)
{
    MessageBox(NULL, L"hijacked again", L"alert", MB_OK);

    return 0;
}


BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    HANDLE threadHandle;
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        threadHandle = CreateThread(NULL, 0, DoMagic2, NULL, 0, NULL);
        CloseHandle(threadHandle);
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

## 引用
1. https://github.com/Flangvik/SharpDllProxy
2. https://docs.microsoft.com/en-us/windows/win32/dlls/dllmain

