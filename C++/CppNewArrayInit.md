---
title: 利用new申请内存时初始化问题
date: 2022-08-24 15:08:22
tags: C++
categories: C/C++
description: C++利用new申请内存时初始化问题。
---



```c++
#include <iostream>

using namespace std;

int main()
{
    int *arrInu = new int[1024];
    for(int idx = 0; idx < 1024; idx++){
        arrInu[idx] = idx;
    }
    delete []arrInu;
    arrInu = nullptr;

    //数组不会初始化为0
    int *arrIno = new int[1024];
    for(int idx = 0; idx < 1024; idx++){
        cout << arrIno[idx] << " ";
    }
    cout << endl;
    delete []arrIno;
    arrIno = nullptr;

    //数组会初始化为0
    int *arrIno0 = new int[1024]();
    for(int idx = 0; idx < 1024; idx++){
        cout << arrIno0[idx] << " ";
    }
    cout << endl;
    delete []arrIno0;
    arrIno0 = nullptr;  

    return 0;
}
```

