---
title: 利用libcurl下载文件
date: 2021-12-06 14:36:04
tags: libcurl
categories: C/C++
description: 利用libcurl下载文件。
---



```c++
#include <iostream>

//下载文件数据接收函数
size_t dlReqReply(void *buffer, size_t size, size_t nmemb, void *user)
{
    FILE *fp = (FILE *)user;
    size_t sizeWrite = fwrite(buffer, size, nmemb, fp);
    return sizeWrite;
}

//http GET请求文件下载
CURLcode dlCurlGetReq(const std::string &url, std::string filename)
{
    FILE *fp = fopen(filename.c_str(), "wb");
    if(NULL == fp){
        LOG_ERROR("Open file<{}> failed.", filename);
        return CURL_LAST;
    }

    CURLcode curlCode = CURL_LAST;
    //curl初始化
    CURL *curl = curl_easy_init();
    if (curl)
    {
        //设置curl的请求头
        struct curl_slist* header_list = NULL;
        header_list = curl_slist_append(header_list, "User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0) like Gecko");
        curl_easy_setopt(curl, CURLOPT_HTTPHEADER, header_list);

        //不接收响应头数据0代表不接收 1代表接收
        curl_easy_setopt(curl, CURLOPT_HEADER, 0);

        //设置请求的URL地址
        curl_easy_setopt(curl, CURLOPT_URL, url.c_str());

        //设置请求重定向
        curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);

        /*设置最大重定向次数*/
        curl_easy_setopt(curl, CURLOPT_MAXREDIRS, 9L);

        //设置ssl验证
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, false);
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYHOST, false);

        //CURLOPT_VERBOSE的值为1时，会显示详细的调试信息
        curl_easy_setopt(curl, CURLOPT_VERBOSE, 0);

        curl_easy_setopt(curl, CURLOPT_READFUNCTION, NULL);

        //设置数据接收函数
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, &dlReqReply);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);

        curl_easy_setopt(curl, CURLOPT_NOSIGNAL, 1);

        //设置超时时间
        curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT, 6); // set transport and time out time
        curl_easy_setopt(curl, CURLOPT_TIMEOUT, 6);

        // 开启请求
        curlCode = curl_easy_perform(curl);
    }
    // 释放curl
    curl_easy_cleanup(curl);
    //释放文件资源
    fclose(fp);

    return curlCode;
}

int main()
{
    std::string strUrl = "";
	std::string strPicturePath = "xiaodaoaa.jpg";

    if(CURLE_OK != dlCurlGetReq(strUrl, strPicturePath)){
        LOG_ERROR("Download failed<{}>.", strUrl);
    }
    
    return 0;
}
```

