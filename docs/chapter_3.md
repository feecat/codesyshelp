# 3 各类总线的专用设置

## 3.1 总线设备的添加与版本更新

在codesys中，项目下添加的设备为第二个PLC，Device下添加的设备为总线。

您也可以在官方help上查到关于各设备的具体信息：[https://help.codesys.com/webapp/f_device_editors;product=codesys](https://help.codesys.com/webapp/f_device_editors;product=codesys){target=_blank}

设备中有一些隐藏的参数，如EtherCAT的DCSyncInWindow参数，在工具-选项，设备编辑器中勾选如下选项以使能额外的设备配置选项卡：
 

## 3.2 EtherCAT的专用设置

系统设置：

- Linux SL/Raspberry pi，不需要特殊的设置。不过若Task Jitter较高可能造成DC同步报警，可修改DCInSyncWindow参数，一般建议改大到500us。

- ControlWin，需要安装 [WinPcap](https://www.winpcap.org/install/bin/WinPcap_4_1_3.exe){target=_blank} 软件才可以运行EtherCAT，且其不具有实时性，不可以运行SoftMotion。

- ControlRTE，需要确保网卡已经安装了专用驱动，您应该可以在设备管理器的网络适配器中看到网卡为CoDeSys Gigabit Network。若开DC和不开DC时发包数量不一致，则需要考虑重装RTE。
	

项目设置：

EtherCAT总线设备分为两个，EtherCAT Master和EtherCAT Master SoftMotion，它们除了FrameAtTaskStart参数外完全一致，带SoftMotion的默认为TRUE，关闭时每个PLC周期结束后才发EtherCAT帧。开启时在每个周期开始时发EtherCAT帧。开启后发送和接收的数据会有延迟，可能会导致超采样出问题，但能提供优异的实时性。

EtherCAT总线依赖设备描述文件，您需要先在工具-设备管理器中安装设备，安装后的设备即可插入在EtherCAT Master下。EtherCAT还有设备树的规范，若您插入的设备是耦合器（如倍福EK1100），则EK1100下挂的模块都需要在EK1100设备上添加。之后的设备在EtherCAT Master下添加。

当您创建EtherCAT Master设备时，会自动创建EtherCAT_Task任务。该任务的周期由EtherCAT Master设备的Cycle Time参数指定，请勿手动修改。此外，需要选择EtherCAT网卡，并建议勾选自动重启从站。

EtherCAT也支持扫描，添加EtherCAT Master并选择网卡后，登陆一次设备并下载PLC。无需启动，此时在EtherCAT Master设备上右键-扫描设备 即可进行扫描。

## 3.3 Profinet的专用设置

根据经验来说，CODESYS的Profinet需求大多数都跑在RT模式，通讯周期一般在1到32MS。由于EPOS的功能块只在博图软件里闭源，所以PTP带轴的实现也极少。大多数情况下，CODESYS的PN需求都是旧设备改造、总线测试或协议转接。

作为PN主站时，操作和EtherCAT非常相似，添加PN主站、扫描或手动添加设备，登录即可。PN不需要专用硬件就可实现星形连接，每个设备都有独立的、在一个网段下的IP。


PLC作为PN从站时，Windows系统中需要将防火墙打开，Linux系统中需要编辑配置文件，在最下面增加：
```
[SysEthernet]
QDISC_BYPASS=1
Linux.ProtocolFilter=3
```

CAUTION: PLC作为PN从站时，建议固定地址而不是使用EnableSetIpAndMask。如果您需要可变的IP，确保由上位指定的IP不会与其它网口在同一网段下。

## 3.4 Modbus RTU的专用设置

在Linux系统中，需编辑配置文件指定COM端口分配。例如：
```
[SysCom]
Linux.Devicefile.1=/dev/ttyAMA0
Linux.Devicefile.2=/dev/ttyUSB0
```

也可以直接归纳到ttyUSB中自动排序，例如：`Linux.Devicefile=/dev/ttyUSB`，这会自动把ttyUSB0映射到COM1，以此类推。

在Windows系统中，请在设备管理器中查看具体的COM端口。

Modbus RTU的设备是Modbus下Modbus Serial Port下的Modbus COM，基于COM端口的通讯。



## 3.5 Modbus TCP的专用设置

ModbusTCPSlave Parameters中有个参数Unit-ID，可以理解为RTU的从站ID。标准协议中已移除此校验，默认置为0xFF。但某些旧设备仍然保留该校验，可手动改为0x01或设备特定的ID。
	
除了标准的Modbus TCP slave设备外，还可以在其下添加Modbus Slave, COM Port，即TCP转RTU方案，可以用协议转换器或某些带RTU扩展的TCP模块。


## 3.6 Ethernet/IP的专用设置

EIP需要注意连接路径，有些设备连接路径不一定是设备描述文件中的。连接路径无法扫描，必须手动输入，不能有任何错误。EIP也可以不使用设备描述文件，而是通过标准设备添加自定义连接路径来连接。

## 3.7 Canopen的专用设置

一般来说，Linux上的Canopen会通过SocketCAN，使用CANable或MCP2515实现。而Windows上一般使用PCAN USB转换器实现。这两种都只支持标准CAN协议，最大波特率1M。

在Linux（Raspberry Pi）中，需要做如下修改：

编辑配置文件，在末尾增加如下行（4.5.0.0之后不再需要）：
```
[CmpSocketCanDrv]
ScriptPath=/opt/codesys/scripts/
ScriptName=rts_set_baud.sh
```
此外，can驱动加载可能晚于运行时，可以关闭自启`sudo systemctl disable codesyscontrol`，延迟启动codesys服务。编辑/etc/rc.local，在exit 0之前插入：
```
sleep 5
sudo service codesyscontrol restart
```

而在windows中，若使用PCAN USB转换器，则需要编辑配置文件，在`[ComponentManager]`下添加一行**Component.X=CmpPCANBasicDrv**（将X更改为顺序的编号）

## 3.8 TCP UDP的专用设置

对于TCP、UDP的底层我们不再重复介绍，有需要的童鞋可以在搜索网站上查一下相关资料。在PLC的实际应用中，我们经常把TCP作为可检测、可靠的UDP使用，即把TCP当作收发包而不是流模式使用。为了达成这一点，我们一般在一个PLC周期内对一个IP、端口只做一次读和/或写操作。如果在一个周期内多次写包，则可能在协议层被自动粘包。此外，单个包的大小超过一定数量也会被自动拆包，需要在应用层做粘包操作。

TCP和UDP都是使用NBS库实现，在CODESYS中分为两个库，一个是官方库（Net Base Services），另一个是CAA库（CAA Net Base Services）。CAA的全称为CoDeSys Automation Alliance，现在已经很少提及，但代码非常稳定。本文档的TCP和UDP以CAA库实现，官方库的实现方法会有细微的区别。

TCP、UDP的通讯实现都是纯代码的，这意味着不需要额外的授权，同样的代码可以在汇川、施耐德等基于CODESYS的PLC上通用。

CAUTION: 官方库和CAA库的命名空间（Namespace）都是NBS，所以不建议在一个项目中同时使用两个库。如果一定要使用，需要手动修改命名空间。


