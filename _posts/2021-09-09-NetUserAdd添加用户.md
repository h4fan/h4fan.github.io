---
layout: post
title:  api添加用户和本地组
tags: [编程]
---

# 声明
本文所涉及程序均在本地运行测试，用于学习目的，请勿用于非法用途。

# 前言
利用NetUserAdd添加用户，NetLocalGroupAddMembers添加用户到本地组。  

前几天看了一篇文章，调用系统api来添加用户，可以绕过某些防护，于是本地进行实验。

# 程序分析
程序见文末。
根据文章，添加用户需要使用NetUserAdd。搜索了一下，好心的微软给了一个完整的程序。参加链接1。直接复制下来就可以用。  

由于我们是本地使用，不需要域，所以稍作修改。`servername`参数为`NULL`时，指向的是本地计算机。所以我们将其默认设置为`NULL`。就可以完成添加用户的操作。  
下一步我们需要添加用户到组，需要使用NetLocalGroupAddMembers函数，继续搜索，发现微软没有给示例代码。/差评。

就需要我们自己来写了。
```cpp
    // add to group
    dwLevel = 3;
    DWORD totalentries = 1;
    LPCWSTR groupname = L"Administrators";
    LOCALGROUP_MEMBERS_INFO_3 lmi[1];
    lmi[0].lgrmi3_domainandname = argv[1];
    nStatus = NetLocalGroupAddMembers(NULL,
        groupname,
        dwLevel,
        (LPBYTE)&lmi,
        totalentries);
```
简单说明一下碰到的问题。
`LPCWSTR`是wchar_t * ,刚开始直接用char * ，导致找不到组。 后面使用L成功。 
有个疑惑的地方是，LOCALGROUP_MEMBERS_INFO_3中的lgrmi3_domainandname是`<DomainName>\<AccountName>`格式，直接用用户名也可以成功。  
buf需要是数组，结果创建了个长度为1的数组，成功运行。  
如果编译之后，运行提示缺少dll，可以参考链接3修改。`项目 -> 配置属性->C/C++->代码生成->运行库 :选择/MT`。


# 运行
使用管理员权限打开cmd，运行，可以成功。某字会拦截，某容和某家没有提示。

# 其它
关于lgrmi3_domainandname可以不输入domainname的问题，网上搜索了一下，可以看到链接5的804行，LsapSplitNames函数中对names进行了判断是否存在\，所以存在可以不用输入domainname的情况。
```cpp
         Ptr = wcschr(Names[i].Buffer, L'\\');
         if (Ptr == NULL)
         {
             AccountLength = Names[i].Length / sizeof(WCHAR);
 
             AccountsBuffer[i].Length = Names[i].Length;
             AccountsBuffer[i].MaximumLength = AccountsBuffer[i].Length + sizeof(WCHAR);
             AccountsBuffer[i].Buffer = MIDL_user_allocate(AccountsBuffer[i].MaximumLength);
```

# 源代码
```cpp
#ifndef UNICODE
#define UNICODE
#endif
#pragma comment(lib, "netapi32.lib")

#include <stdio.h>
#include <windows.h> 
#include <lm.h>

int wmain(int argc, wchar_t* argv[])
{
    USER_INFO_1 ui;
    DWORD dwLevel = 1;
    DWORD dwError = 0;
    NET_API_STATUS nStatus;

    if (argc != 3)
    {
        fwprintf(stderr, L"Usage: %s UserName Password \n", argv[0]);
        exit(1);
    }
    //
    // Set up the USER_INFO_1 structure.
    //  USER_PRIV_USER: name identifies a user, 
    //    rather than an administrator or a guest.
    //  UF_SCRIPT: required 
    //
    ui.usri1_name = argv[1];
    ui.usri1_password = argv[2];
    ui.usri1_priv = USER_PRIV_USER;
    ui.usri1_home_dir = NULL;
    ui.usri1_comment = NULL;
    ui.usri1_flags = UF_SCRIPT;
    ui.usri1_script_path = NULL;
    //
    // Call the NetUserAdd function, specifying level 1.
    //
    //nStatus = NetUserAdd(argv[1],
    //    dwLevel,
    //    (LPBYTE)&ui,
    //    &dwError);
    nStatus = NetUserAdd(NULL,
        dwLevel,
        (LPBYTE)&ui,
        &dwError);
    //
    // If the call succeeds, inform the user.
    //
    if (nStatus == NERR_Success)
        fwprintf(stderr, L"User %s has been successfully added on %s\n",
            argv[1], L"local");
    //
    // Otherwise, print the system error.
    //
    else
        fprintf(stderr, "A system error has occurred: %d\n", nStatus);

    // add to group
    dwLevel = 3;
    DWORD totalentries = 1;
    LPCWSTR groupname = L"Administrators";
    LOCALGROUP_MEMBERS_INFO_3 lmi[1];
    lmi[0].lgrmi3_domainandname = argv[1];
    nStatus = NetLocalGroupAddMembers(NULL,
        groupname,
        dwLevel,
        (LPBYTE)&lmi,
        totalentries);
    //
    // If the call succeeds, inform the user.
    //
    if (nStatus == NERR_Success)
        fwprintf(stderr, L"User %s has been successfully added to %s\n",
            argv[1], groupname);
    //
    // Otherwise, print the system error.
    //
    else
        fprintf(stderr, "A system error has occurred: %d\n", nStatus);

    return 0;
}
```

## 引用
1. https://docs.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuseradd
2. https://docs.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netlocalgroupaddmembers
3. https://blog.csdn.net/Richarll/article/details/6591550
4. https://doxygen.reactos.org/d9/df7/local__group_8c.html#ae85c698e825c3f6b9eb711809ad1a03b
5. https://doxygen.reactos.org/d9/d99/dll_2win32_2lsasrv_2lookup_8c.html#a91a7fd0133e555e0953cf4f5508cff5b
