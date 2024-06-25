---
title: RabbitMQ生产者及消费者示例
date: 2021-12-06 16:24:59
tags: C/C++
categories: C/C++
description: RabbitMQ生产者及消费者示例。
---



# Common.h

```c++
#ifndef COMMON_H
#define COMMON_H

#include <string>

struct RabbitMqInfo
{
    std::string strIp;          //RabbitMq服务端ip
    int nPort;                  //端口
    std::string strUser;        //用户名
    std::string strPwd;         //密码
    std::string strExchange;    // 交换机名
    std::string strRoutingKey;  // 路由名
    int nChannelId;             // 频道号

    void reset()
    {
        strIp = "";
        nPort = 6005;
        strUser = "";
        strPwd = "";
        strExchange = "";
        strRoutingKey = "";
        nChannelId = 1;
    }

    RabbitMqInfo& operator=(const RabbitMqInfo& o)
    {
        if (this != &o)
        {
            strIp = o.strIp;
            nPort = o.nPort;
            strUser = o.strUser;
            strPwd = o.strPwd;
            strExchange = o.strExchange;
            strRoutingKey = o.strRoutingKey;
            nChannelId = o.nChannelId;
        }

        return *this;
    }
};

#endif

```

# RabbitmqConsumer.h

```c++
#ifndef RABBITMQ_CONSUMER_H
#define RABBITMQ_CONSUMER_H

#include <Windows.h>
#include <iostream>
#include <atomic>
#include <time.h>
#include <amqp.h>
#include <amqp_tcp_socket.h>
#include "Common.h"

class CRabbitmqConsumer
{
public:
    explicit CRabbitmqConsumer(const RabbitMqInfo& info)
    {
        m_info = info;
    }

    ~CRabbitmqConsumer()
    {
        Stop();
        DisConnect();
    }

    bool Start()
    {
        m_bStop = false;
        if (Connect() && PreConsumer())
        {
            Consumer();
        }

        return true;
    }

    bool Stop()
    {
        m_bStop = true;
        return true;
    }

    //连接
    bool Connect()
    {
        bool bRet(false);
        std::string strErr("");

        std::cout << "[Connect] Begin connect ..." << std::endl;

        do
        {
            // 1.创建一个新连接
            m_conn = amqp_new_connection();
            if (!m_conn)
            {
                strErr = "amqp_new_connection failed";
                break;
            }

            // 2.创建一个新socket
            amqp_socket_t *socket = amqp_tcp_socket_new(m_conn);
            if (!socket)
            {
                strErr = "amqp_tcp_socket_new failed";
                break;
            }

            // 3.打开socket，设置IP、port等
            if (amqp_socket_open(socket, m_info.strIp.c_str(), m_info.nPort) != AMQP_STATUS_OK)
            {
                strErr = "amqp_socket_open failed";
                break;
            }

            // 4.登录服务器
            amqp_rpc_reply_t rpc_reply = amqp_login(
                m_conn,
                "/",
                0,
                AMQP_DEFAULT_FRAME_SIZE,
                0,
                AMQP_SASL_METHOD_PLAIN,
                m_info.strUser.c_str(),
                m_info.strPwd.c_str()
            );
            if (0 != getErr(rpc_reply, "amqp_login", strErr))
            {
                break;
            }

            // 5.打开通道，关联conn和channel
            if (!amqp_channel_open(m_conn, m_info.nChannelId))
            {
                strErr = "amqp_channel_open failed";
                break;
            }

            bRet = true;
        } while (false);

        std::cout << "[Connect] bRet : " << bRet << std::endl;
        std::cout << "[Connect] : " << strErr << std::endl;

        return bRet;
    }

    //断开连接
    bool DisConnect()
    {
        bool bRet(false);
        std::string strErr("");

        do
        {
            // 1. 关闭channel
            amqp_rpc_reply_t rpc_reply = amqp_channel_close(m_conn, m_info.nChannelId, AMQP_REPLY_SUCCESS);
            if(0 != getErr(rpc_reply, "amqp_channel_close", strErr))
            {
                break;
            }

            // 2.关闭连接
            rpc_reply = amqp_connection_close(m_conn, AMQP_REPLY_SUCCESS);
            if (0 != getErr(rpc_reply, "amqp_connection_close", strErr))
            {
                break;
            }

            // 3.销毁连接
            if (amqp_destroy_connection(m_conn) != AMQP_STATUS_OK)
            {
                strErr = "amqp_destroy_connection failed";
                break;
            }
            m_conn = NULL;

            bRet = true;
        } while (false);

        std::cout << strErr << std::endl;

        return bRet;
    }

    // 将队列绑定到指定交换机上，预消费
    bool PreConsumer()
    {
        bool bRet(false);
        std::string strErr("");

        std::cout << "[PreConsumer] : PreConsumer begin ..." << std::endl;

        do
        {
            // 1.声明队列
            amqp_queue_declare_ok_t *r = amqp_queue_declare(
                m_conn,
                m_info.nChannelId,
                amqp_empty_bytes,
                0, 0, 1, 1, amqp_empty_table);
            amqp_rpc_reply_t rpc_reply = amqp_get_rpc_reply(m_conn);
            if (0 != getErr(rpc_reply, "amqp_queue_declare", strErr))
            {
                break;
            }

            // 2.分配队列
            amqp_bytes_t queueName = amqp_bytes_malloc_dup(r->queue);
            if (queueName.bytes == NULL)
            {
                strErr = "amqp_bytes_malloc_dup failed";
                break;
            }

            // 3.队列与交换机绑定
            amqp_queue_bind_ok_t *res = amqp_queue_bind(
                m_conn,
                m_info.nChannelId,
                queueName,
                amqp_cstring_bytes(m_info.strExchange.c_str()),
                amqp_cstring_bytes(m_info.strRoutingKey.c_str()),
                amqp_empty_table);
            (void)(res);
            rpc_reply = amqp_get_rpc_reply(m_conn);
            if (0 != getErr(rpc_reply, "amqp_queue_bind", strErr))
            {
                break;
            }

            // 4.设置消费消息参数
            amqp_basic_consume_ok_t *res2 = amqp_basic_consume(
                m_conn,
                m_info.nChannelId,
                queueName,
                amqp_empty_bytes,
                /*no_local*/ 0,	//true if the server should not deliver to this consumer messages published on this channel’s connection
                                //true：mq服务器不应将当前频道上的消息发送到此consumer
                /*no_ack*/ 1,	//是否需要确认消息后再从队列中删除消息
                /*exclusive*/ 0,
                amqp_empty_table);
            (void)(res2);
            rpc_reply = amqp_get_rpc_reply(m_conn);
            if (0 != getErr(rpc_reply, "amqp_basic_consume", strErr))
            {
                break;
            }

            bRet = true;
        } while (false);

        std::cout << "[PreConsumer] bRet : " << bRet << std::endl;
        std::cout << "[PreConsumer] : " << strErr << std::endl;

        return bRet;
    }

    // 消费
    void Consumer()
    {
        std::string strErr{ "" };

        std::cout << "[Consumer] m_bStop : " << m_bStop << std::endl;
        while (!m_bStop)
        {
            // 1.释放buffers
            amqp_maybe_release_buffers(m_conn);
            // 2.定义信封载体
            amqp_envelope_t envelope;
            struct timeval tv = {0};
            tv.tv_sec = 2;
            // 3.把收到的相关信息放入信封中envelope
            amqp_rpc_reply_t rpc_reply = amqp_consume_message(m_conn, &envelope, &tv, 0);
            if (0 != getErr(rpc_reply, "amqp_basic_consume", strErr))
            {
                if (rpc_reply.library_error == AMQP_STATUS_TIMEOUT)
                {
                    //std::cout << "[Consumer] errno : AMQP_STATUS_TIMEOUT" << std::endl;
                    continue;
                }
                else if (rpc_reply.library_error == AMQP_STATUS_SOCKET_ERROR)
                {
                    std::cout << "[Consumer] errno : AMQP_STATUS_SOCKET_ERROR" << std::endl;
                    break;
                }
                else
                {
                    std::cout << "[Consumer] errno : Unknow" << std::endl;
                    break;
                }
            }

            //4.获取消息内容
            //if (envelope.message.properties._flags & AMQP_BASIC_CONTENT_TYPE_FLAG)
            if (AMQP_RESPONSE_NORMAL == rpc_reply.reply_type)
            {
                if(0 < envelope.message.body.len)
                {
                    std::cout << "[Consumer] msg len : " << envelope.message.body.len << std::endl;
                    std::cout << "[Consumer] message : " << (const char *)envelope.message.body.bytes << std::endl;
                }
            }

            // 5.毁掉信封
            amqp_destroy_envelope(&envelope);
        }

        // 6. 释放连接信息
        DisConnect();
    }

    //获取错误信息
    int getErr(const amqp_rpc_reply_t& reply, const std::string& strModule, std::string& strErr)
    {
        char szErrMsg[1024] { 0 };
        switch (reply.reply_type)
        {
        case AMQP_RESPONSE_NORMAL:
            return 0;
        case AMQP_RESPONSE_NONE:
            sprintf(szErrMsg, "%s: missing RPC reply type!\n", strModule.c_str());
            break;
        case AMQP_RESPONSE_LIBRARY_EXCEPTION:
            sprintf(szErrMsg, "%s: %s\n", strModule.c_str(), amqp_error_string2(reply.library_error));
            break;
        case AMQP_RESPONSE_SERVER_EXCEPTION:
            switch (reply.reply.id)
            {
            case AMQP_CONNECTION_CLOSE_METHOD:
            {
                amqp_connection_close_t *m = (amqp_connection_close_t *)reply.reply.decoded;
                sprintf(szErrMsg, "%s: server connection error %d, message: %.*s\n",
                    strModule.c_str(),
                    m->reply_code,
                    (int)m->reply_text.len, (char *)m->reply_text.bytes);
                break;
            }
            case AMQP_CHANNEL_CLOSE_METHOD:
            {
                amqp_channel_close_t *m = (amqp_channel_close_t *)reply.reply.decoded;
                sprintf(szErrMsg, "%s: server channel error %d, message: %.*s\n",
                    strModule.c_str(),
                    m->reply_code,
                    (int)m->reply_text.len, (char *)m->reply_text.bytes);
                break;
            }
            default:
                sprintf(szErrMsg, "%s: unknown server error, method id 0x%08X\n", strModule.c_str(), reply.reply.id);
                break;
            }
            break;
        }
        strErr = szErrMsg;

        return -1;
    }


private:
    bool m_bConnect { false };
    RabbitMqInfo m_info;
    amqp_connection_state_t m_conn;
    std::atomic_bool m_bStop { false };             // 设置线程退出
};

#endif

```

# main.cpp

```c++
#include <iostream>
#include <RabbitmqConsumer.h>

using namespace std;

int main()
{
    cout << "RabbitmqConsumer start ..." << endl;

    RabbitMqInfo info;
    info.strIp = "127.0.0.1";
    info.nPort = 5672;
    info.strUser = "admin";
    info.strPwd = "admin";
    info.strExchange = "ExchangeFanout";
    info.strRoutingKey = "";
    info.nChannelId = 1;

    CRabbitmqConsumer consumer(info);
    consumer.Start();

    cout << "RabbitmqConsumer exit ..." << endl;

    return 0;
}

```



# RabbitmqProducer.h

```c++
#ifndef RABBITMQ_PRODUCER_H
#define RABBITMQ_PRODUCER_H

#include <iostream>
#include <atomic>
#include <thread>
#include <chrono>
#include <ctime>
#include <amqp.h>
#include <amqp_tcp_socket.h>
#include "Common.h"

class CRabbitmqProducer
{
public:
    explicit CRabbitmqProducer(const RabbitMqInfo& info)
    {
        m_info = info;
    }

    ~CRabbitmqProducer()
    {
        Stop();
        DisConnect();
    }

    bool Start()
    {
        m_bStop = false;

        if (Connect() && PreProducer())
        {
            Producer();
        }

        return true;
    }

    bool Stop()
    {
        m_bStop = true;

        return true;
    }

    //连接
    bool Connect()
    {
        bool bRet{ false };
        std::string strErr{ "" };
        do
        {
            // 1.创建一个新连接
            m_conn = amqp_new_connection();
            if (!m_conn)
            {
                strErr = "amqp_new_connection failed";
                break;
            }

            // 2.创建一个新socket
            amqp_socket_t *socket = amqp_tcp_socket_new(m_conn);
            if (!socket)
            {
                strErr = "amqp_tcp_socket_new failed";
                break;
            }

            // 3.打开socket，设置IP、port等
            if (amqp_socket_open(socket, m_info.strIp.c_str(), m_info.nPort) != AMQP_STATUS_OK)
            {
                strErr = "amqp_socket_open failed";
                break;
            }

            // 4.登录服务器
            amqp_rpc_reply_t rpc_reply = amqp_login(
                        m_conn,
                        "/",
                        1,
                        AMQP_DEFAULT_FRAME_SIZE,
                        30,
                        AMQP_SASL_METHOD_PLAIN,
                        m_info.strUser.c_str(),
                        m_info.strPwd.c_str()
                        );
            if (0 != getErr(rpc_reply, "amqp_login", strErr))
            {
                break;
            }

            // 5.打开通道，关联conn和channel
            if (!amqp_channel_open(m_conn, m_info.nChannelId))
            {
                strErr = "amqp_channel_open failed";
                break;
            }

            bRet = true;
        } while (false);

        std::cout << strErr << std::endl;

        return bRet;
    }

    //断开连接
    bool DisConnect()
    {
        bool bRet{ false };
        std::string strErr{ "" };
        do
        {
            // 1. 关闭channel
            amqp_rpc_reply_t rpc_reply = amqp_channel_close(m_conn, m_info.nChannelId, AMQP_REPLY_SUCCESS);
            if (0 != getErr(rpc_reply, "amqp_channel_close", strErr))
            {
                break;
            }

            // 2.关闭连接
            rpc_reply = amqp_connection_close(m_conn, AMQP_REPLY_SUCCESS);
            if (0 != getErr(rpc_reply, "amqp_connection_close", strErr))
            {
                break;
            }

            // 3.销毁连接
            if (amqp_destroy_connection(m_conn) != AMQP_STATUS_OK)
            {
                strErr = "amqp_destroy_connection failed";
                break;
            }
            m_conn = NULL;

            bRet = true;
        } while (false);

        std::cout << strErr << std::endl;

        return bRet;
    }

    // 预发布
    bool PreProducer()
    {
        return true;
    }

    //发布消息
    void Producer()
    {
        std::string msg{ "" };
        std::string strErr{ "" };

        while (!m_bStop)
        {
            time_t now = time(0);
            char* dt = ctime(&now);
            msg = dt;

            amqp_basic_properties_t props;
            props._flags = AMQP_BASIC_CONTENT_TYPE_FLAG | AMQP_BASIC_DELIVERY_MODE_FLAG;
            props.content_type = amqp_cstring_bytes("text/plain");
            props.delivery_mode = 2; // persistent delivery mode 持久化模式

            //发布消息
            if (0 != amqp_basic_publish(m_conn, 1, amqp_cstring_bytes(m_info.strExchange.c_str()), amqp_cstring_bytes(m_info.strRoutingKey.c_str()), 0, 0, &props, amqp_cstring_bytes(msg.c_str())))
            {
                if (0 != getErr(amqp_get_rpc_reply(m_conn), "amqp_basic_publish", strErr))
                {
                    break;
                }
            }

            std::cout << msg << std::endl;

            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        }
        // 3. 释放连接信息
        DisConnect();
    }

    //获取错误信息
    int getErr(const amqp_rpc_reply_t& reply, const std::string& strModule, std::string& strErr)
    {
        char szErrMsg[1024]{ 0 };
        switch (reply.reply_type)
        {
        case AMQP_RESPONSE_NORMAL:
            return 0;
        case AMQP_RESPONSE_NONE:
            sprintf(szErrMsg, "%s: missing RPC reply type!\n", strModule.c_str());
            break;
        case AMQP_RESPONSE_LIBRARY_EXCEPTION:
            sprintf(szErrMsg, "%s: %s\n", strModule.c_str(), amqp_error_string2(reply.library_error));
            break;
        case AMQP_RESPONSE_SERVER_EXCEPTION:
            switch (reply.reply.id)
            {
            case AMQP_CONNECTION_CLOSE_METHOD:
            {
                amqp_connection_close_t *m = (amqp_connection_close_t *)reply.reply.decoded;
                sprintf(szErrMsg, "%s: server connection error %d, message: %.*s\n",
                        strModule.c_str(),
                        m->reply_code,
                        (int)m->reply_text.len, (char *)m->reply_text.bytes);
                break;
            }
            case AMQP_CHANNEL_CLOSE_METHOD:
            {
                amqp_channel_close_t *m = (amqp_channel_close_t *)reply.reply.decoded;
                sprintf(szErrMsg, "%s: server channel error %d, message: %.*s\n",
                        strModule.c_str(),
                        m->reply_code,
                        (int)m->reply_text.len, (char *)m->reply_text.bytes);
                break;
            }
            default:
                sprintf(szErrMsg, "%s: unknown server error, method id 0x%08X\n", strModule.c_str(), reply.reply.id);
                break;
            }
            break;
        }
        strErr = szErrMsg;
        std::cout << strErr << std::endl;

        return -1;
    }


private:
    bool m_bConnect{ false };
    RabbitMqInfo m_info;
    amqp_connection_state_t m_conn;
    std::atomic_bool m_bStop{ false };             // 设置线程退出
};

#endif // !RABBITMQ_PRODUCER_H


```



# main.cpp

```c++
#include <iostream>
#include <RabbitmqProducer.h>

using namespace std;

int main()
{
    RabbitMqInfo info;
    info.strIp = "127.0.0.1";
    info.nPort = 5672;
    info.strUser = "admin";
    info.strPwd = "admin";
    info.strExchange = "ExchangeFanout";
    info.strRoutingKey = "";
    info.nChannelId = 1;

    CRabbitmqProducer producer(info);
    producer.Start();

    return 0;
}

```

