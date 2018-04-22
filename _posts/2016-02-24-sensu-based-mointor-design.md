---
ID: 71
post_title: 基于Sensu的监控设计
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/02/24/sensu-based-mointor-design/
published: true
post_date: 2016-02-24 12:17:36
---
优秀的成熟监控框架有很多，像Zabbix, Sensu, 小米的Open-falcon, InfluxData等，它们都有各自的优势所在；从我的角度来看监控方案的问题不在于框架之间的优劣，而在于如何将监控对象能够统一起来纳入到整体的监控方案，实现组织内集中式的监控，这才是监控方案的价值和难度所在。硬件资源、操作系统、进程端口、数据、API通常对这些方面做好有效的监控，结合良好的流程控制，就可以逐步做到平台的稳定以及防患于未然。

然而现实和理想总是背道而驰，Shell, Java, Python + Crontab的监控方式和凌晨爬起处理现实问题的习惯更容易被认同和接受。认知和理念才是最根本的问题！

<strong>监控平台设计</strong>
[embeddoc url="http://hua-fei.com.cn/wp-content/uploads/2016/02/BI监控系统架构设计.docx" height="890px" viewer="microsoft"]

<strong>Sensu监控安装配置</strong>
[embeddoc url="http://hua-fei.com.cn/wp-content/uploads/2016/02/Sensu安装文档.docx" height="890px" viewer="microsoft"]