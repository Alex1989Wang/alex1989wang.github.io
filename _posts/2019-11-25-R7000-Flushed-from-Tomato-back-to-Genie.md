---
layout: post
title: R7000 Flushed from Tomato back to Genie
date: 2019-11-25 09:32:21 +0800
tags: router
categories: Chinese
---

Tomato代理设置更新的不变，让我决定从Tomato刷到梅林。所以需要将router还原回出厂的固件。

# 准备

 - [R7000-back-to-ofw.trx](http://tomato.groov.pl/download/K26ARM/Netgear%20R-series%20back%20to%20OFW/)

## 开始

 - 彻底清除NVRAM

     Administration->Configuration

     等待重启

 - 更新固件

     Administration->Upgrade
     R7000-back-to-ofw.trx

     等待固件上传和刷机
     等待重启

 - 登录路由器

     www.routerlogin.net
     admin
     password

     已经变成了netgear的genie

 - 清除备份

     Advanced
     Administration->Backup Settings->回到出厂设置
     等待重启

 - 完成


