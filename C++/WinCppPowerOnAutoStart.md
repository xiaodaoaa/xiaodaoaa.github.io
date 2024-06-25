---
title: Windows C++ 程序开机自启
date: 2021-12-20 11:52:19
tags: C/C++
categories: C/C++
description: Windows平台下C++程序设置开机自启动
---

```c++
#include <windows.h>

//设置当前程序开机启动
void AutoPowerOn()
{
    HKEY hKey;
 
    //1、找到系统的启动项，win7以后权限管理越来越严格，非管理员权限运行的程序是无法写入到HKEY_LOCAL_MACHINE下的，推荐添加启动项到当前用户
    if (RegOpenKeyEx(HKEY_CURRENT_USER, _T("SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run"), 0, KEY_ALL_ACCESS, &hKey) == ERROR_SUCCESS) ///打开启动项       
    {
        //2、得到本程序自身的全路径
        TCHAR strExeFullDir[MAX_PATH];
        GetModuleFileName(NULL, strExeFullDir, MAX_PATH); 
 
        //3、判断注册表项是否已经存在
        TCHAR strDir[MAX_PATH] = {};
        DWORD nLength = MAX_PATH;
        long result = RegGetValue(hKey, nullptr, _T("ApplicationName"), RRF_RT_REG_SZ, 0, strDir, &nLength); 
 
        //4、已经存在
        if (result != ERROR_SUCCESS || _tcscmp(strExeFullDir, strDir) != 0)
        {
            //5、添加一个子Key,并设置值，"ApplicationName"是应用程序名字（不加后缀.exe） 
            RegSetValueEx(hKey, _T("ApplicationName"), 0, REG_SZ, (LPBYTE)strExeFullDir, (lstrlen(strExeFullDir) + 1)*sizeof(TCHAR)); 
 
            //6、关闭注册表
            RegCloseKey(hKey);
        }
    }
}
 
 
//取消当前程序开机启动
void CanclePowerOn()
{
    HKEY hKey;

    //1、找到系统的启动项  
    if (RegOpenKeyEx(HKEY_CURRENT_USER, _T("SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run"), 0, KEY_ALL_ACCESS, &hKey) == ERROR_SUCCESS)
    {
        //2、删除值
        RegDeleteValue(hKey, _T("ApplicationName")); 
 
        //3、关闭注册表
        RegCloseKey(hKey);
    }
}
```

