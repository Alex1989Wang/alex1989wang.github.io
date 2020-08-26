---
layout: page
title: About
date: 2017-09-09 23:36:50
permalink: /about/
---

# 基本信息

| 姓名 | 邮箱 | 博客 | GitHub |
| :---: | :---: | :---: | :---: | 
| 王江 | alex1989wang@gmail.com | http://www.awsomejiang.com | https://github.com/Alex1989Wang |

# 自我评价

- 对编程有强烈的兴趣。
- 思维敏捷、缜密，有很强的自学能力。
- 性格沉稳、踏实，乐于团队协作。
- 喜欢挑战和专研技术；热爱`Vim`编辑器。

# 专业技能 

- App开发技能：
	- 能熟练使用 Objective-C/Swift语言进行开发; 熟悉 C、Python 语言;
	- 了解常用的算法和数据结构
	- 能够熟练使用代码和`Interface Builder`进行App界面的开发；对绝大部分UI控件十分熟悉；能够使用`Quartz2D`绘制自定义图形；[熟练使用`Core Animation`设计完成复杂动画](https://alex1989wang.gitbooks.io/core-animation/content/)；
	- 熟悉iOS开发中常用的设计模式；能熟练应用`MVC`模式，使用`通知`、`KVO`、`代理`、`单例`等机制；
	- 熟练使用CoreData框架对应用数据本地持久化；
	- [熟悉Objective-C内存管理机制](http://www.awsomejiang.com/2018/01/15/Memory-Management-For-iOS-Apps-No-2/)及Runtime、Runloop;
	- 掌握多线程编程技术；能够熟练使用`GCD`和`NSOperation`；
	- 掌握网络请求的发送和常用数据类型`（JSON和XML）`解析；
	- 熟练掌握iOS端多种数据持久化机制：如`Core Data`、`Sqlite`;
	- 掌握`Instruments`使用；能够对App进行`Time Profile`、`Memory Usage Profile`和`Leaks`的检查；
	- 掌握`XCTest`框架；能够编写单元测试和UI测试；
	- 熟练使用第三方框架：`AFNetworking`、`Masonry`、`SDWebImage`等；
- 其他相关技能：
	- 无障碍阅读英文文档，观看WWDC；
	- 熟练掌握SQL数据库查询语言和Sqlite数据库使用；
	- 使用`Vim`编辑器快速、准确编辑代码；
	- 熟悉`Git`和`SVN`版本控制器工作流程；

# 开源

- [JWWaveView](https://github.com/Alex1989Wang/JWWaveView)
- [JWInfiniteCollectionView](https://github.com/Alex1989Wang/JWInfiniteCollectionView)

# 工作经验

----
- 公司简介 

| 时间 | 公司名称 | 职位 | 规模 | 离职原因 |
| :---: | :---: | :---: | :---: | :---: |
| 2018.04 - 至今 | 成都品果科技有限公司 | iOS高级开发工程师 | 150左右 | 无 |

- 项目简介
    - Camera360。该项目是主打美颜自拍的相机应用。
    - App下载地址：[Camera360](https://apps.apple.com/cn/app/camera360-ultimate/id443354861)。

- 项目职责:
    - 负责开发和维护各个产品的广告聚合SDK。聚合多家广告商提供的广告类型，采用Protocol封装统一的接口
    - 采用MVVM开发美颜、美妆模块交互UI和数据持久化
    - 负责部分相机引擎接口(聚焦模式、对焦、变焦等)
    - 维护和开发IAP内购模块，使用A/B测试框架来测试不同内购商品组合的收益
    - 维护拼图酱等系列产品，开发长图拼接等模块
    - 结合GCD和KVO优化相册列表页的展示效率

----
- 公司简介 

| 时间 | 公司名称 | 职位 | 规模 | 离职原因 |
| :---: | :---: | :---: | :---: | :---: |
| 2016.08 - 2018.02 | 广州市久邦数码有限责任公司（[Sungy Mobile Limited.](http://www.3g.net.cn/)) | iOS开发工程师 | 500左右 | 回老家成都了 |

- 项目简介
    - GO Live。该项目是主打国外市场的一款直播软件。
    - App下载地址：[itunes app store - go live](https://itunes.apple.com/app/id1296697241?mt=8)。

- 项目主要开发内容：
    - 独立开发礼物赠送模块：包括礼物资源的下载、自定义FlowLayout的礼物面板、礼物特效使用lottie框架来排队播放；优化大礼物的性能；
    - 对礼物缓存做了一套动态加载更新的策略；设计downloader下载器，使用信号量来控制下载的并发总量，避免下载时长过长而超时失败；
    - 采用Core Data对通知消息进行存储，是既上个App采用Sqlite的一次不同尝试；方便轻量级迁移；
    - 为整个App设计了一套应用[通用的弹窗动画展示和排队的机制](http://www.awsomejiang.com/2017/09/27/design-roubust-alert-views/)；为App设计了一套房间内的弹出面板系统，有效协调和管理不同面板的弹出、收起和重叠展示；
    - 采用对布局cache和mark dirty来高效地实现公屏消息的宽高随外框调整而调整；
    - 通知和KVO有机结合完成房间状态的维护和同步；
    - 对核心逻辑使用XCTest框架编写了单元测试；
    - 独立开发房间内赠送宝箱的功能，其中使用KVO来降低模块耦合；

- 其他

独立开发、维护整个`App`的消息`handler`、礼物赠送和特效显示逻辑、宝箱功能、公屏展示、整套应用内弹窗系统、房间内的弹出面板机制、假直播房间、用户包裹功能。

----

- 公司简介 

| 时间 | 公司名称 | 职位 | 规模 | 离职原因 |
| :---: | :---: | :---: | :---: | :---: |
| 2015.03 – 2016.03 | 新东方广州学校国外部 | 托福口语教师 | 500左右 | 转行做技术 |

- 工作职责
	- 完成托福口语部分的课程设计和教学。


