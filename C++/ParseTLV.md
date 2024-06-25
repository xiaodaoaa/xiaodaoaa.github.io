---
title: TLV协议格式解析
date: 2021-12-24 17:14:40
tags: [C/C++, TLV]
categories: C/C++
description: TLV协议格式报文解析。
---



# TLV协议简介

[BER](https://baike.baidu.com/item/BER/19940289)编码一种，[ASN.1](https://baike.baidu.com/item/ASN.1)标准，全称Type（类型），Length（长度），Value（值）。

IS-IS数据通信领域中，tlv三元组： Type**-length-value**（**TLV**）。T、L字段的长度往往固定（通常为1～4bytes），V字段长度可变。顾名思义，T字段表示[报文](https://baike.baidu.com/item/报文)类型，L字段表示报文长度、V字段往往用来存放报文的内容。

TLV 是一种可变的格式，其中：

- `T` 可以理解为 `Tag` 或 `Type` ，用于标识标签或者编码格式信息；
- `L` 定义数值的长度；
- `V` 表示实际的数值。

`T` 和 `L` 的长度固定，一般是2或4个字节，`V` 的长度由 `Length` 指定。



# 大小端

要正确的解析对方发来的数据除了统一数据格式之外还要统一`字节序`。`字节序是指多字节数据在计算机内存中存储或者网络传输时各字节的存储顺序`。字节序一般分为**大端**和**小端**。

**大端模式（Big-Endian）**： 高位字节放在内存的低地址端，低位字节排放在内存的高地址端。

**小端模式（Little-Endian）**： 低位字节放在内存的低地址端，高位字节放在内存的高地址端。

下面举个例子，要把0x1234 存放在 0x0000 地址处，那么大端模式和小端模式存放方式如下：

|  地址  | 小端模式 | 大端模式 |
| :----: | :------: | :------: |
| 0x0000 |   0x34   |   0x12   |
| 0x0001 |   0x12   |   0x34   |



# 报文解析

1. 读取 Tag（或Type），指针偏移4；
2. 读取 Length，指针偏移4；
3. 根据得到的长度读取 Value，若为 int、char、short、long 类型，指针偏移；若值为字符串，读取后指针偏移 Length；
4. 重复上述三步，继续读取后面的 TLV 单元。

```c++
#include <iostream>
#include <string>
#include <cstdio>

using namespace std;

struct Message
{
    int age;
    string name;

    Message() {
        this->age = 0;
        this->name = "";
    }
};

int main()
{
    Message info;

    char pchTmpBuff[128] = {0};
    char chBuff[] = {0x01, 0x00, 0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x45, 0x00, 0x00, 0x00,
                     0x02, 0x00, 0x00, 0x00, 0x0A, 0x00, 0x00, 0x00, 0x78, 0x69, 0x61, 0x6F,
                     0x20, 'm', 'i', 'n', 'g', 0x00};

    char *pchHead = chBuff;
    char *pchTail = chBuff + sizeof(chBuff);

    char *pchTmp = pchHead;
    while(pchTmp < pchTail)
    {
        int tag = 0;
        int len = 0;

        //取出tag/type
        memcpy(&tag, pchTmp, 4);
        pchTmp += 4;
        cout << "tag:" << tag << endl;

        //取出数据长度len
        memcpy(&len, pchTmp, 4);
        pchTmp += 4;
        cout << "len:" << len << endl;

        //取出value
        switch (tag) {
        case 0x00000001:
            memcpy(&info.age, pchTmp, len);
            break;

        case 0x00000002:
            memset(pchTmpBuff, 0x00, sizeof(pchTmpBuff));
            memcpy(pchTmpBuff, pchTmp, len);
            info.name = pchTmpBuff;
            break;

        default:
            break;
        }

        pchTmp += len;
    }

    cout << "info.age:" << info.age << endl;
    cout << "info.name:" << info.name << endl;

    return 0;
}
```

