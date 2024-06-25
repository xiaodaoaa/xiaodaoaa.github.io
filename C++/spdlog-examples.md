---
title: Spdlog简易封装及示例
date: 2021-12-06 14:44:01
tags: [Spdlog,C/C++]
categories: C/C++
description: Spdlog简易封装及示例。
---



# SpdlogHeader.h

```c++
#ifndef SPDLOGHEADER_H
#define SPDLOGHEADER_H
#include <memory>
#include <spdlog.h>
#include <sinks/rotating_file_sink.h>
#include <fmt/bin_to_hex.h>

class BaseLog
{
private:
    BaseLog()=default;
    ~BaseLog(){
        // Release all spdlog resources, and drop all loggers in the registry.
        // This is optional (only mandatory if using windows + async log).
        //spdlog::shutdown();
    }
public:
    static BaseLog *getInstance() {
        static BaseLog instance;
        return &instance;
    }

    //初始化日志，路径使用locale编码
    //如: QString("日志.log").toLocal8Bit().toStdString()
    void init(const std::string &path){
        // Create a file rotating logger with 10mb size max and 5 rotated files.
        logPtr = spdlog::rotating_logger_mt("Spdlog", path, 10*1048576, 5);

        //设置日志记录级别
        logPtr->set_level(spdlog::level::trace);

        //设置格式
        /* [%Y-%m-%d %H:%M:%S.%e]:时间
         * [%l]:日志级别,[%t]:线程,[%s]:文件,[%#]:行号,[%!]:函数,[%v]:实际文本 */
        logPtr->set_pattern("[%Y-%m-%d %H:%M:%S.%e] [%L] [%t] [%s:%!:%#] %v");

        //设置当出发 err 或更严重的错误时立刻刷新日志到  disk
        logPtr->flush_on(spdlog::level::trace);

        //spdlog::flush_every(std::chrono::seconds(3));
    }

    auto logger() {
        return logPtr;
    }

private:
    std::shared_ptr<spdlog::logger> logPtr;
};

#define SpdlogInit(path)     BaseLog::getInstance()->init(path)
#define SPDLOG_BASE(logger, level, ...) (logger)->log(spdlog::source_loc{__FILE__, __LINE__, __func__}, level, __VA_ARGS__)
#define LOG_TRACE(...)     SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::trace, ##__VA_ARGS__)
#define LOG_DEBUG(...)     SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::debug, __VA_ARGS__)
#define LOG_INFO(...)      SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::info, __VA_ARGS__)
#define LOG_WARN(...)      SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::warn, __VA_ARGS__)
#define LOG_ERROR(...)     SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::err, __VA_ARGS__)
#define LOG_CRITICAL(...)  SPDLOG_BASE(BaseLog::getInstance()->logger(), spdlog::level::critical, __VA_ARGS__)

#endif // SPDLOGHEADER_H

```



# main.cpp

```c++
#include "SpdlogHeader.h"
#include <future>
#include <string>
#include <thread>
#include <chrono>
#include <atomic>
#include <iostream>
#include <vector>
#include <QByteArray>
#include <QString>
#include <sstream>

using namespace std;

int main()
{
    SpdlogInit("logs/TSpdlog.log");

    LOG_DEBUG(666);

    const char *cstrMsg = "hello, world~";
    LOG_DEBUG("[MSG] {}", cstrMsg);


    LOG_DEBUG("{}", spdlog::to_hex(cstrMsg, cstrMsg+5));

    std::vector<char> vecBuff;
    vecBuff.insert(vecBuff.begin(), cstrMsg, cstrMsg+strlen(cstrMsg));
    LOG_DEBUG("{}", spdlog::to_hex(vecBuff));
    LOG_DEBUG("{:n}", spdlog::to_hex(std::begin(vecBuff), std::begin(vecBuff) + strlen(cstrMsg)));

    std::vector<char> buf;
    for (int i = 0; i < 80; i++){
        buf.push_back(static_cast<char>(i & 0xff));
    }

    // Log binary data as hex.
    // Many types of std::container<char> types can be used.
    // Iterator ranges are supported too.
    // Format flags:
    // {:X} - print in uppercase.
    // {:s} - don't separate each byte with space.
    // {:p} - don't print the position on each line start.
    // {:n} - don't split the output to lines.
    // {:a} - hexdump style
    LOG_DEBUG("{}", spdlog::to_hex(buf));
    LOG_DEBUG("{:n}", spdlog::to_hex(std::begin(buf), std::begin(buf) + 10));
    LOG_DEBUG("{:X}", spdlog::to_hex(buf));
    LOG_DEBUG("{:Xs}", spdlog::to_hex(buf));
    LOG_DEBUG("{:Xsp}", spdlog::to_hex(buf));
    LOG_DEBUG("{:a}", spdlog::to_hex(buf));
    LOG_DEBUG("{:a}", spdlog::to_hex(buf, 20));

    QByteArray byteArry;
    byteArry.append(buf.data(), buf.size());
    LOG_DEBUG("{}", spdlog::to_hex(byteArry));


    std::atomic_bool bQuit{false};
    bQuit = true;
    auto logPrint = [&](std::string msg){
        LOG_INFO(msg);
        while(!bQuit) {
            LOG_TRACE("Support for int: {0:d};  hex: {0:x};  oct: {0:o}; bin: {0:b}", 42);
            LOG_DEBUG("Easy padding in numbers like {:08d}", 12);
            LOG_INFO("Welcome to spdlog version {}.{}.{}  !", SPDLOG_VER_MAJOR, SPDLOG_VER_MINOR, SPDLOG_VER_PATCH);
            LOG_WARN("Support for floats {:03.2f}", 1.23456);
            LOG_ERROR("Positional args are {1} {0}..", "too", "supported");
            LOG_CRITICAL("{:>8} aligned, {:<8} aligned", "right", "left");
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }

        cout << "The log-thread will exit ..." << endl;
    };

    auto logFuture = std::async(logPrint, "Begin test spdlog...");

    getchar();
    bQuit = true;
    logFuture.wait();

    cout << "The app will exit." << endl;

    return 0;
}

```



# 程序输出：

```bash
[2021-12-06 14:50:30.183] [D] [3860] [main.cpp:main:19] 666
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:22] [MSG] hello, world~
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:25] 
0000: 68 65 6c 6c 6f
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:29] 
0000: 68 65 6c 6c 6f 2c 20 77 6f 72 6c 64 7e
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:30]  68 65 6c 6c 6f 2c 20 77 6f 72 6c 64 7e
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:46] 
0000: 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
0020: 20 21 22 23 24 25 26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35 36 37 38 39 3a 3b 3c 3d 3e 3f
0040: 40 41 42 43 44 45 46 47 48 49 4a 4b 4c 4d 4e 4f
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:47]  00 01 02 03 04 05 06 07 08 09
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:48] 
0000: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F
0020: 20 21 22 23 24 25 26 27 28 29 2A 2B 2C 2D 2E 2F 30 31 32 33 34 35 36 37 38 39 3A 3B 3C 3D 3E 3F
0040: 40 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:49] 
0000: 000102030405060708090A0B0C0D0E0F101112131415161718191A1B1C1D1E1F
0020: 202122232425262728292A2B2C2D2E2F303132333435363738393A3B3C3D3E3F
0040: 404142434445464748494A4B4C4D4E4F
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:50] 
000102030405060708090A0B0C0D0E0F101112131415161718191A1B1C1D1E1F
202122232425262728292A2B2C2D2E2F303132333435363738393A3B3C3D3E3F
404142434445464748494A4B4C4D4E4F
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:51] 
0000: 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f  ................................
0020: 20 21 22 23 24 25 26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35 36 37 38 39 3a 3b 3c 3d 3e 3f   !"#$%&'()*+,-./0123456789:;<=>?
0040: 40 41 42 43 44 45 46 47 48 49 4a 4b 4c 4d 4e 4f                                                  @ABCDEFGHIJKLMNO
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:52] 
0000: 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13  ....................
0014: 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25 26 27  ............ !"#$%&'
0028: 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35 36 37 38 39 3a 3b  ()*+,-./0123456789:;
003C: 3c 3d 3e 3f 40 41 42 43 44 45 46 47 48 49 4a 4b 4c 4d 4e 4f  <=>?@ABCDEFGHIJKLMNO
[2021-12-06 14:50:30.185] [D] [3860] [main.cpp:main:56] 
0000: 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
0020: 20 21 22 23 24 25 26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35 36 37 38 39 3a 3b 3c 3d 3e 3f
0040: 40 41 42 43 44 45 46 47 48 49 4a 4b 4c 4d 4e 4f
[2021-12-06 14:50:30.185] [I] [21008] [main.cpp:operator ():62] Begin test spdlog...

```

