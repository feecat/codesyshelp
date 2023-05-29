# 7 电气基础

## 7.1 写在结尾

首页推荐阅读中有一本北岛李工的书，介绍中有一点我觉得特别好，就是在书中的实例讲解中，作者都会首先介绍电气图纸，因为只有明白电气接线，程序讲解才有意义。本章节会介绍电气基础知识，供读者参考。我偏向于实践，可能会和理论有些出入，若有错误欢迎指正。同时，电气工程师不应局限于PLC编程和电气图纸，也应当了解实际接线问题、电子器件、程序运行的知识，会极大增加阅历，好找BUG。

## 7.2 供电、接地和载流量

一般的10~100KW自动化设备，当需要380V供电时，建议选用TT（三相五线）系统，单独打接地柱（铜包钢接地棒或石墨接地体）。若条件不允许也可用TN-S系统（厂区共用接地），但需确认PE有正确连接到接地网络。

自动化设备的长期运行平均电流一般远低于各设备额定电流之和，单设备的瞬发峰值电流可能超过额定电流的2倍。在选用电缆时，需考虑长度、载流量和超额运行时间。用于供电的电缆宜选用RVV（多股多芯软导线）或BVR（多股单芯软导线），用于主供电的电缆一般比设计的额定载流量高一个级别（或除以降额系数0.85）。在设备内部的信号电缆选用RV（多股单芯软导线）或RVVP（多股多芯屏蔽线）电缆。具体到载流量，个人经验如下：

| 单芯截面积 | 载流量 | 备注 |
| ---- | ---- | ---- |
| <=2.5平 | 截面积*10A | 满载建议在有空调的电气柜内 |
| 4~10平 | 截面积*7A | 常用于小设备供电或伺服电机，建议RVV |
| 16~35平 | 截面积*4A | 常用于中型设备主供电 |
| 50~120平 | 截面积*3A | 一般选单芯BVR，建议架空，电柜需特殊进线 |
| >=150平 | 截面积*2A | 与电压、端子和铺设方式关系大，载流量仅供参考 |

**实际载流量根据设备负载情况、设备类型有极大的不同，某些情况下100平的BVR线架空可以负载400A以上，另外一些情况下2.5平RVV线穿管长期负载只有16A。故该表仅供参考。** 线缆选型并不是非黑即白，截面过大也并非都错。建议在设备上用电流钳或电流探头卡一下实际电流，有条件的还可以拿红外相机看一下温升，可对线缆选型有进一步的了解。

三相供电任取一相，与零线构成220V交流供电，给AC-DC转换器（如明伟NDR-240-24），即可得到24V直流供电。在干扰较大的场所下，还可以对AC-DC的输出加装滤波器。一般来说，AC-DC的输出负极建议在电柜内单点接地，相当于给定参考电平，可以极大降低干扰。

带IO的PLC或总线模块有两路或多路供电，它们的总线供电和逻辑供电是隔离的。在大多数情况下，都接入一个电源即可。只有当跨设备时，总线供电应由主站负责，IO供电应由设备本地负责。一般情况下，不应以抗干扰为由设置两个电源，但可以抗干扰为由增加直流滤波器。若多个电源同时供电，应当增加冗余供电模块（如明纬DRDN40-24）而不是将两个电源输出直接连接。

## 7.3 逻辑电平和负载类型

PNP和NPN是三极管的类型，在电气设计中也指信号的逻辑电平类型。当使用PNP类型时，信号线悬空、接0V为无信号，接24V为有信号。当使用NPN类型时，信号线悬空、接24V为无信号，接0V为有信号。若信号存在交叉情况，例如传感器为PNP、设备为NPN，则中间需要光耦或继电器转接。某些设备的公共点可以自定义（接线柱带COM标识），可以兼容PNP/NPN（内部结构为双向光耦）。

NO和NC是触点或信号类型，分别为常开（Normal-Open）和常闭（Normal-Close），在处理继电器、接触器时常用。但在一些特殊设备，如时间继电器、逻辑控制器或外部设备上，最好结合时序图查看。NO-NC与PNP-NPN无关，具体电平取决于触点另一端的接线。


LED指示灯、电阻丝为阻性负载，不需要特殊处理。电机、线圈为感性负载，需要做续流或RC保护。一般情况下，PLC或IO模块的直流晶体管型输出（例如西门子DCDCDC或倍福EL2809）都具有续流保护，带指示灯的电磁阀和继电器也都内置续流保护。不建议用继电器型PLC或外置继电器直接驱动感性负载。若负载电流超过PLC输出能力，可考虑通过PLC放大板（MOS管阵列）或接触器扩展。

## 7.4 数字模拟、总线信号和共地

PNP和NPN信号都是数字信号，只有0和1两种状态。作为输出时，最大频率一般在1KHz。作为输入时，部分模块可配置输入滤波时间，一般建议4MS滤波，即最大125Hz，程序里还可以用TOF或TON功能进一步滤波。部分模块有专用PWM IO（例如倍福EL2502），频率可达100KHz，用于脉冲控制或PWM控制，输出能力较小。

模拟信号由模拟正和模拟负组成，一般使用±10V、0~10V电压信号和0~20mA、4~20mA电流信号，分辨率从8位到16位，响应时间在0.5~20ms。当信号线距离较长时宜选用RVVP屏蔽线，并使用电流信号。在一定情况下模拟信号的负极也可以接地以增强抗干扰能力。

新设备上总线宜选用基于网线的总线，例如EtherCAT、ProfiNet、EtherNET/IP和Modbus TCP，基于网线的总线很容易排除线缆问题，并且网线有极强的抗干扰能力。当一定需要RS485和CAN总线时，宜选用双绞屏蔽线，电柜内可不用阻抗匹配。超过一定长度后需要终端电阻、隔离器和使用阻抗匹配的线缆。

若设备上有驱动器、变频器等控制电机的设备，电机线需采用屏蔽线，电柜内应设有屏蔽轨道，可剥开外皮用EMC屏蔽电缆夹固定。部分驱动器上也有固定卡扣，应当确认屏蔽层连接牢固。驱动器、变频器等设备说明书一定有关于EMC解决方案的声明，请仔细阅读。

关于接地和信号可以参考 [杭和分散控制系统（DCS）接地作业标准](http://www.gkwo.net/wenku/show-57774.html){target=_blank} 以及 [和利时DCS硬件系统培训](http://www.gkwo.net/wenku/show-62122.html){target=_blank} 。工控资料窝上还有不少手册，可以延伸阅读。

## 7.5 电气设备选型

现代化的电气设备应当摒弃专用分线端子和冗长的继电器中继理念，而是选用紧凑型IO板、弹片式接线端子和标准化的重载插头。

- 端子推荐：魏德米勒ZDU/WAGO 281/菲尼克斯PT（都是弹片式）
- IO推荐：倍福EK1100/汇川GL20/魏德米勒UR20（紧凑型卡片式，可根据需求插入不同数量的卡片）
- 重载推荐：哈丁/魏德米勒/唯恩 10B/16B/24B，标准产品好找替代。有高密度要求选HDC/HDD/HEE系列。
- 特殊接头推荐：泰科/菲尼克斯，主要用于伺服电机、编码器和防水接头。
- 按钮推荐：施耐德/RAFI/GK
- 线缆推荐：IGUS/LAPP/HELUKABEL
- 三色灯推荐：PATLITE/WERMA
- 其它弱电产品推荐：西门子/施耐德/WAGO
- 安全产品推荐：pilz

## 7.6 工具及软件

- 万用表是电气工程师必不可少的工具，推荐一个钳形表（FLUKE 317/UT210E）和一个普通表（FLUKE 15B/UT191T）搭配使用。
- 当需要进行电子方面的测试和维修时，需要一个热风枪+焊台（WY815P）、可调直流电源（UTP1305S）。
- 当需要监控RS485信号或测量波形时，需要示波器（PicoScope 2206B/SDS1102X）。
- 五金工具可选Wera/STANLEY/SATA，工具是消耗品，选对工具比质量好的工具更重要。


软件方面，常用以下几个：

- [Google](https://www.google.com/){target=_blank} 、 [BaiDu](https://www.baidu.com/){target=_blank} 、 [TaoBao](https://www.taobao.com/){target=_blank} 、 [NewBing](https://www.bing.com/search?q=Bing+AI&showconv=1){target=_blank} ，善用搜索工具
- [ModbusTool](https://github.com/ClassicDIY/ModbusTool){target=_blank} ，Modbus RTU和TCP的测试，带日志
- [PROFINET Commander](https://profinetcommander.com/){target=_blank} ，PN测试工具
- [VSCode](https://code.visualstudio.com/){target=_blank} 、 [Eclipse](https://www.eclipse.org/){target=_blank} 、 [Notepad++](https://notepad-plus-plus.org/downloads/){target=_blank} ，编辑文件
- [WindTerm](https://github.com/kingToolbox/WindTerm){target=_blank} 、 [Putty](https://www.putty.org/){target=_blank} ，SSH工具
- [FileZilla](https://filezilla-project.org/){target=_blank} 、 [Baby FTP Server](https://www.pablosoftwaresolutions.com/html/baby_ftp_server.html){target=_blank} ，FTP客户端及服务器
- [7-ZIP](https://www.7-zip.org/){target=_blank} ，压缩文件工具
- [WizTree](https://diskanalyzer.com/){target=_blank} ，目录大小分析工具
- [Snipaste](https://www.snipaste.com/){target=_blank} ，截图工具
- [Wu10Man](https://github.com/WereDev/Wu10Man){target=_blank} ，禁用win10更新
- [HEU_KMS_Activator](https://github.com/zbezj/HEU_KMS_Activator){target=_blank} ，KMS激活工具
- [sokit](https://github.com/sinpolib/sokit/releases/tag/v1.3.20111130){target=_blank} ，TCP+UDP调试工具
- [HslCommunication](https://github.com/dathlin/HslCommunication){target=_blank} ，各种通讯库合集（付费）
- [RoboDK](https://robodk.com/){target=_blank} ，工业机器人仿真（付费）
- [Acrobat Pro](https://www.adobe.com/acrobat/acrobat-pro.html){target=_blank} 、 [PDF-XChange Pro](https://www.pdf-xchange.eu/pdf-xchange-pro/index.htm){target=_blank} ，PDF工具（付费）
- [KICAD](https://www.kicad.org/){target=_blank} 、 [LCEDA](https://lceda.cn/){target=_blank} ，PCB设计
- [Dash](https://plotly.com/dash/){target=_blank} 、 [Node-RED Electron](https://github.com/feecat/electron-node-red){target=_blank} ，数据可视化


