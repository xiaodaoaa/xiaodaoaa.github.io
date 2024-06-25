---
title: SSH免密登录
date: 2021-12-03 11:19:04
tags: linux
categories: linux
description: VSCode SSH免密登录。
---

# SSH免密登录
## **1、用户电脑生成SSH密钥和公钥：**

```bash
$ ssh-keygen -t rsa -b 4096 
$ ls
config  id_rsa  id_rsa.pub  known_hosts  known_hosts.old
```
## **2、将公钥拷贝至服务器：**

```bash
$ ssh-copy-id -i id_rsa.pub user@host
```
## **3、完成。**
