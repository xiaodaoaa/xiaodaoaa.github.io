---
title: librdkafka消费者示例
date: 2021-12-06 16:40:58
tags: C/C++
categories: C/C++
description: librdkafka消费者示例。
---



# KafkaConsumerClient.h

```c++
#ifndef KAFKACONSUMERCLIENT_H
#define KAFKACONSUMERCLIENT_H

#include <iostream>
#include <string>
#include <cstdlib>
#include <cstdio>
#include <csignal>
#include <cstring>
#include <list>
#include <rdkafkacpp.h>
#include <vector>
#include <fstream>
#include <atomic>

using namespace std;

class KafkaConsumerClient {
public:
    explicit KafkaConsumerClient(const std::string &brokers,
                                 const std::string &topics,
                                 std::string groupid,
                                 int32_t nPartition,
                                 int64_t offset);

	virtual ~KafkaConsumerClient();
	//初始化
	bool Init();
	//开始获取消息
    void Run(int timeout_ms);
    //开始
    void Start();
	//停止
	void Stop();
    //获取运行状态
    bool isRun();
    //退出
    bool Quit();

    //获取Offset
    uint64_t GetLastOffset();

private:
	void Msg_consume(RdKafka::Message* message, void* opaque);
    void ParseJsonMsg(const char *strJsonMsg, const int len);

private:
	std::string m_strBrokers;
	std::string m_strTopics;
	std::string m_strGroupid;
    int64_t m_nLastOffset{0};
    RdKafka::Consumer *m_pKafkaConsumer{nullptr};
    RdKafka::Topic    *m_pTopic{nullptr};
    int64_t           m_nCurrentOffset{RdKafka::Topic::OFFSET_STORED};
    int32_t           m_nPartition{0};
    atomic_bool m_bRun{false};
    atomic_bool m_bQuit{false};
};
#endif // KAFKACONSUMERCLIENT_H

```



# KafkaConsumerClient.cpp

```c++
#include "KafkaConsumerClient.h"
#include "ImageInfoVector.h"
#include "json.h"
#include "SpdlogHeader.h"

#include <fstream>
#include <iostream>
#include <unistd.h>

KafkaConsumerClient::KafkaConsumerClient(
        const std::string &brokers,
        const std::string &topics,
        std::string groupid,
        int32_t nPartition,
        int64_t offset)	:
    m_strBrokers(brokers),
	m_strTopics(topics),
	m_strGroupid(groupid),
    m_nCurrentOffset(offset),
    m_nPartition(nPartition)
{

}

KafkaConsumerClient::~KafkaConsumerClient()
{
    Quit();
}

bool KafkaConsumerClient::Init() {
	std::string errstr;
	RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
	if (!conf) {
		std::cerr << "RdKafka create global conf failed" << endl;
		return false;
	}
	/*设置broker list*/
	if (conf->set("metadata.broker.list", m_strBrokers, errstr) != RdKafka::Conf::CONF_OK) {
		std::cerr << "RdKafka conf set brokerlist failed ::" << errstr.c_str() << endl;
	}
	/*设置consumer group*/
	if (conf->set("group.id", m_strGroupid, errstr) != RdKafka::Conf::CONF_OK) {
		std::cerr << "RdKafka conf set group.id failed :" << errstr.c_str() << endl;
	}
	std::string strfetch_num = "10240000";
	/*每次从单个分区中拉取消息的最大尺寸*/
	if (conf->set("max.partition.fetch.bytes", strfetch_num, errstr) != RdKafka::Conf::CONF_OK) {
		std::cerr << "RdKafka conf set max.partition failed :" << errstr.c_str() << endl;
	}
	/*创建kafka consumer实例*/ //Create consumer using accumulated global configuration.
	m_pKafkaConsumer = RdKafka::Consumer::create(conf, errstr);
	if (!m_pKafkaConsumer) {
		std::cerr << "failed to ceate consumer" << endl;
	}
	std::cout << "% Created consumer " << m_pKafkaConsumer->name() << std::endl;
	delete conf;
	/*创建kafka topic的配置*/
	RdKafka::Conf *tconf = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);
	if (!tconf) {
		std::cerr << "RdKafka create topic conf failed" << endl;
		return false;
	}
	if (tconf->set("auto.offset.reset", "smallest", errstr) != RdKafka::Conf::CONF_OK) {
		std::cerr << "RdKafka conf set auto.offset.reset failed:" << errstr.c_str() << endl;
	}
	/*
	* Create topic handle.
	*/
	m_pTopic = RdKafka::Topic::create(m_pKafkaConsumer, m_strTopics, tconf, errstr);
	if (!m_pTopic) {
		std::cerr << "RdKafka create topic failed :" << errstr.c_str() << endl;
	}
	delete tconf;
	/*
	* Start consumer for topic+partition at start offset
	*/
    //for (int i = 0; i < 10; i++)
	{
        RdKafka::ErrorCode resp = m_pKafkaConsumer->start(m_pTopic, m_nPartition, m_nCurrentOffset);
		if (resp != RdKafka::ERR_NO_ERROR) {
			std::cerr << "failed to start consumer : " << errstr.c_str() << endl;
		}
	}

	return true;
}
void KafkaConsumerClient::Msg_consume(RdKafka::Message* message, void* opaque) {
    (void)(opaque);
	switch (message->err()) {
	case RdKafka::ERR__TIMED_OUT:
		break;

	case RdKafka::ERR_NO_ERROR:
		/* Real message */
        //std::cout << "Read msg at offset " << message->offset() << std::endl;
		if (message->key()) {
			std::cout << "Key: " << *message->key() << std::endl;
		}

        //printf("partition:%d, offset:%ld\n", message->partition(), message->offset());
        //printf("%.*s\n", static_cast<int>(message->len()), static_cast<const char *>(message->payload()));

		m_nLastOffset = message->offset();

        ParseJsonMsg(static_cast<const char *>(message->payload()), static_cast<int>(message->len()));
		break;

	case RdKafka::ERR__PARTITION_EOF:
		/* Last message */
		cout << "Reached the end of the queue, offset: " << m_nLastOffset << endl;
		//Stop();
		break;
	case RdKafka::ERR__UNKNOWN_TOPIC:
	case RdKafka::ERR__UNKNOWN_PARTITION:
		std::cerr << "Consume failed: " << message->errstr() << std::endl;
		Stop();
		break;

	default:
		/* Errors */
		std::cerr << "Consume failed: " << message->errstr() << std::endl;
		Stop();
		break;
    }
}

void KafkaConsumerClient::ParseJsonMsg(const char *strJsonMsg, const int len)
{
    JSONCPP_STRING errs;
    Json::Value root;
    Json::CharReaderBuilder readerBuilder;
    Json::CharReader *jsonReader = readerBuilder.newCharReader();
    if (jsonReader->parse(strJsonMsg, strJsonMsg + len, &root, &errs)) {
		//TODO...
    }

    delete jsonReader;
}

void KafkaConsumerClient::Run(int timeout_ms) {
	RdKafka::Message *msg = NULL;
    while (true) {
        if(m_bRun){
            msg = m_pKafkaConsumer->consume(m_pTopic, m_nPartition, timeout_ms);
            Msg_consume(msg, NULL);
            delete msg;
            m_pKafkaConsumer->poll(0);
        }
        else{
            if(m_bQuit) break;
            sleep(1);
        }
	}

	m_pKafkaConsumer->stop(m_pTopic, m_nPartition);
	m_pKafkaConsumer->poll(1000);

	if (m_pTopic) {
		delete m_pTopic;
		m_pTopic = NULL;
	}

	if (m_pKafkaConsumer) {
		delete m_pKafkaConsumer;
		m_pKafkaConsumer = NULL;
	}

	/*销毁kafka实例*/ //Wait for RdKafka to decommission.
    RdKafka::wait_destroyed(5000);
}

void KafkaConsumerClient::Start()
{
    m_bRun = true;
}

void KafkaConsumerClient::Stop()
{
    m_bRun = false;
}

bool KafkaConsumerClient::isRun()
{
    return m_bRun;
}

bool KafkaConsumerClient::Quit()
{
    Stop();
    m_bQuit = true;
    return true;
}

uint64_t KafkaConsumerClient::GetLastOffset()
{
    return this->m_nLastOffset;
}

```



# main.cpp

```c++
#include "KafkaConsumerClient.h"
#include <stdio.h>

int main()
{
    KafkaConsumerClient *KafkaConsumerClient_ = new KafkaConsumerClient("192.168.0.1:9092", "TEST_TOPIC", "0", 0, RdKafka::Topic::OFFSET_STORED);//OFFSET_BEGINNING,OFFSET_END
    if (!KafkaConsumerClient_->Init())
    {
        fprintf(stderr, "kafka server initialize error\n");
        return -1;
    }

    KafkaConsumerClient_->Run(1000);
    KafkaConsumerClient_->Start();
	
	return 0;
}
```

