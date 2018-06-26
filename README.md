# 基本信息  


姓名： 吴浩&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;性别：&nbsp;&nbsp;男  

手机：18826408875  &nbsp; &nbsp;邮箱：992488670@qq.com  

英语等级:四级 &nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;工作年限：3年   

应聘岗位：Android开发工程师  
# 工作技能
1 熟练掌握 java 开发技术,具有较好的 java 基础，熟悉JVM 的内存回收机制，熟悉java8 HashMap、ArrayList源码等  

2 熟练掌握Android开发技术，熟悉android Handler机制、AsnycTask源码，Activity 加载模式、Android LruCache源码、systemServer启动流程、LocalBroadcastManager源码等

3 熟悉安卓view的绘制流程、事件分发机制和布局优化，熟悉ViewStub工作原理

4 熟悉安卓数据库开发，熟悉安卓SharedPreferences工作原理  

5 熟悉tcp/upd,熟悉http/https工作原理。了解okHttp3工作原理和流程分析

6 熟悉安卓第三方推送服务（小米推送和华为推送），华为推送方案高度结合云之家已有业务  

7 良好的团队协作能力和沟通能力，具有一定的产品思维
 
# 教育背景
时间	&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;学校	&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp;  &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;专业	&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; 学历

2011-2015	&nbsp; &nbsp;广东工业大学	&nbsp; &nbsp;&nbsp; &nbsp;软件工程	&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;本科

# 工作经验
**公司名称**&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;**深圳云之家网络有限公司** &nbsp;&nbsp;&nbsp;&nbsp;**规模**&nbsp;&nbsp;&nbsp;&nbsp;500-1000人  

**工作时间**&nbsp; &nbsp;2015-7至今	&nbsp; &nbsp;&nbsp; &nbsp;**职位**	 &nbsp; &nbsp;Android开发工程师  

**职责**&nbsp; &nbsp;&nbsp; &nbsp;项目组TeamLeader（小组成员9人）  

**汇报对象**&nbsp; &nbsp;云之家副总经理  

**工作内容**  
1 项目组各迭代的任务安排和技术方案制定及资源协调  

2 项目模块各迭代周期的开发任务，设计和维护android项目底层模块和业务 

3 对接云之家开放平台，为云之家开放平台提供基础服务和技术支持  

4 完成云之家通用基础组件的设计和开发，如通用ListView 索引栏和水印布局的设计和实现  

5 优化重构代码，对现有的复杂模块进行重构和优化  

6 对接客服部门，快速定位和解决客户问题  

# 项目经验
### 项目名称&nbsp;&nbsp;&nbsp;&nbsp;人员列表展示性能优化

**时间**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2017-11月  

**项目简介**  
通讯录模块中，需要根据需要，展示人员数据，并且按照字母分栏，并在右侧显示索引条。由于历史原因，展示方案是先从数据库中load出所需要的数据，然后在内存中进行排序，这个过程造成了很多不必要的性能浪费。因此需要进行优化  

**技术方案**  
数据库升级，往数据库中建立拼音字段，这个字段可用于生成索引条和索引栏，同时可以按照这个字段进行数据库排序。listview的adapter改用cursorAdapter

**项目成果**  

SQL排序+CursorAdapter方案，使列表展示速度得到明显的提升。根据数据分析，外部好友列表展示速度提升80%，其他页面展示速度提升在40%以上 

-----


### 项目名称&nbsp;&nbsp;&nbsp;&nbsp;复杂业务重构（人员详情和组织架构模块）

**时间**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2017-3月  

**项目简介**  
随着项目的快速发展，组织服务中重要的两个大模块：人员详情和组织机构因为历史原因，已经不能快速相应业务的快速迭代，增加一个列显示或者排序顺序的变化，都会让原本复杂的代码变得难以维护和增加功能，因此需要进行业务重构  

**技术方案**  
采用MVP+RecycleView+MultiType的设计架构，将各个大模块抽离出小的模块，进行解耦。以后如果需要变更，只需要调整顺序，修改provider或者新增一个provider加入展示列表即可

**项目成果**  
本次重构，将所有的业务进行拆分和组装，在功能变更的时候，可以快速相应需求，而且各个小模块相对独立，不会对以后的业务造成影响，便于验证回归 

-----

### 项目名称&nbsp;&nbsp;&nbsp;&nbsp;通用选人组件设计和重构 

**时间**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2016-7到2016-8  

**项目简介**  
云之家选人组件意在为各个业务模块提供基础选人服务，由于选人组件历经多人开发和维护，造成选人组件与业务耦合度极高，各个业务模块对选人组件的ui有不一样的要求，多个业务实现代码放在组件中，导致选人组件极易出现选人bug，不好扩展和维护。  

**技术方案**  
采用mvp的设计模式，对于人员数据展示采用策略分发的模式，基于业务端决定选人组件的样式和功能的思想，对整个选人组件进行完整重构。 

**项目成果**  
统一各个业务线组件的调用模式。选人组件的重构在业务调用和结果返回更加的简单和方便，提高了开发的效率和代码的稳定性，这个基础设计思想也被ios开发组所采用  

-----

### 项目名称&nbsp;&nbsp;&nbsp;&nbsp;云之家消息推送搭建和优化  

**时间**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2017-4月  

**项目简介**  
由于安卓平台的限制，云之家push的到达率不高，许多用户经常反馈杀掉进程后，聊天消息就收不到了。本次优化，针对不同的手机厂商，设计适合云之家业务的推送方案，提升消息的到达率。  

**技术方案**  
引入小米pushSdk和华为pushSdk，结合长连接websocket，在特定的场景下，不同的手机厂商，透传和非透传push方式切换，提升消息的到达率

**项目成果**  

新的推送方案上线后，极大的提高了消息的到达率，用户反馈消息接收不到的问题大幅度减少。

------
### 项目名称&nbsp;&nbsp;&nbsp;&nbsp;云之家消息分页	  

**时间**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2015-12月  

**项目简介**  
由于历史原因，云之家聊天界面的获取采用的是轮询的方式，在10秒的周期内，根据本地的updateTime去服务端获取最新的消息，导致消息量大时，消息获取和界面绘制卡顿，在极端环境下，最新消息内容无法展现，需要制定一套新的消息获取方案解决这个问题  

**技术方案**  
基于websocket的长连接推送方案，进入聊天页签时，优先展示本地数据，同时根据场景，判断何时需要拉取消息，往新的消息拉取，还是老的消息拉取

**项目成果**  

消息分页方案在1个月内实现并成功上线，解决消息获取缓慢和界面卡顿的情况，并稳定运行 

-----
# 荣誉和奖励
1.深圳云之家网络有限公司2017年精英人才  

2.深圳云之家网络有限公司2016年年度优秀员工  

3.深圳云之家网络有限公司2015届部门优秀纯金   

附:  
github地址：https://github.com/gdutkyle  
简书地址：https://www.jianshu.com/u/25d79c70dcf5