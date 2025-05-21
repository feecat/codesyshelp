# 11 附录：PNDriver

## 11.1 简介

在之前的文章中，我们聊过了EtherCAT主站和ProfiNET从站。EtherCAT从站需要专用ASIC如LAN9253、AX58200、XMC4300等实现，底层还是倍福的SSC。而ProfiNET主站一直都没有开源实现。

西门子官方其实是有ProfiNET主站和从站代码的，叫做PNDriver，与之配套的还有PNConfigLib。目前最新版本为 [PNDriver3.2、PNConfigLib1.5](https://support.industry.siemens.com/cs/document/109974538/) 。

PNDriver一直在更新迭代，但其主要为CP1625客户提供，需要密码才可以解压。它既可以作为主站，也可以作为从站使用。源代码可以跑在CP1625、Windows和Linux上。

## 11.2 解密

在PNDriver的早期版本中，源代码被封装为了ISO文件再经过加密，而最近的版本直接通过ZIP压缩并加密，这导致了一个漏洞可以被利用，基本原理是通过 [bkcrack](https://github.com/kimci86/bkcrack) 检测ZIP中仅压缩且未混淆的文件，通过明文去恢复内部密钥，再通过密钥对ZIP文件解密。

要进行解密，我们需要先在压缩包内找到算法为**ZipCrypto Store**的文件，例如`PNDriver_03.02\contrib\Debian_Packages\pndevdrv-dkms_03.02.00.00.00.01.00.15_amd64.deb`。然后，我们需要知道这个文件的一段明文，deb文件通常以以下字符开头，注意换行符为LF：
```bash
!<arch>
debian-binary
```

之后就可以照着bkcrack的介绍进行密钥恢复了，恢复出来的是内部密钥，只可以将zip压缩包的密码解除，要想获取原始密码还是有一定难度，不在本文讨论范围。拿到源代码后，我们就可以开始测试了。本文之后主要讨论Linux（LinuxCNC）下的ProfiNET主站应用，带一个V90从站跑RT（非实时）。

## 11.3 配置环境和编译

PNDriver的基础文档做的很漂亮，我们主要看Quick Start for Linux Native这部分。我习惯使用VSCode，安装和配置照着文档做就可以。

Linux默认的端口范围不满足，可通过`sudo sh -c 'echo "49152 65535" > /proc/sys/net/ipv4/ip_local_port_range'`临时修改。

建议首次编译`pn_driver\src\examples\ioc\test_app`，ioc表示controller（主站）。test_app中已有部分基础代码，运行之后，通过2、5、9、10来依次选择网卡、启动主站、连接从站。主站可以启动就没问题了，默认的从站是ET200SP我们没有，需要换成V90。

## 11.4 使用PNConfigLib

之前我提到过西门子的一大特色是稳定，另一大特色是臃肿。PNConfigLib就是一个典型。由于GSDML设备描述文件无法直接被PNDriver使用，当存在多个从站时，需要对从站分配IP、检查拓扑结构，这部分的描述文件转换过程被做到了PNConfigLib中，即通过PNConfigLib生成中间层xml文件，再由PNDriver读取并驱动从站。

PNConfigLib的技术栈用的是VC++混合C#和.net 8.0，所以推荐在windows系统下操作。我们需要参考`PNConfigLib_01.05\buildtool\README.md`文档，对源代码进行编译，需要vs2022和.net开发环境。编译之后，可执行文件在`PNConfigLib_01.05\bin\Release\x64\PNConfigLibRunner.exe` 。

编译出来的文件不带界面，还是需要通过命令行操作，操作的文档在`PNConfigLib_01.05\examples\Readme.md`。对于我们测试来说，我们只用basic configuration，即如下指令：
```
.\PNConfigLibRunner -c "..\..\..\examples\01_Basic_Configuration\Configuration.xml" -l "..\..\..\examples\01_Basic_Configuration\ListOfNodes.xml"
```

执行后，将自动新建文件夹`PNConfigLib Output - [日期时间]`，里面的`PROFINET Driver_PN_Driver_1.xml`就是PNDriver需要的中间描述层。

本文的目标是带V90伺服，所以我们还需要将V90的GSDML拷到`examples\GSDMLs`下，并修改01_Basic_Configuration下的两个xml。其中，**Configuration.xml**存放站名、插槽交互，主体内容如下：

```
<Configuration schemaVersion="1.0"
               ConfigurationID="ConfigurationID"
               ConfigurationName="ConfigurationName"
               ListOfNodesRefID="ListOfNodesID"
               xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xmlns="http://www.siemens.com/Automation/PNConfigLib/Configuration"
               xsi:schemaLocation="http://www.siemens.com/Automation/PNConfigLib/Configuration Configuration.xsd">
    <Devices>
        <CentralDevice DeviceRefID="PN_Driver_1">
            <CentralDeviceInterface InterfaceRefID="PN_Driver_1_Interface">
                <EthernetAddresses>
                    <IPProtocol>
                        <SetInTheProject IPAddress="100.168.0.1"
                                         SubnetMask="255.255.255.0"
                                         RouterAddress="100.168.0.100" />
                        <!--<SetDirectlyAtTheDevice/>-->
                    </IPProtocol>
                    <PROFINETDeviceName>
                        <PNDeviceName>pndriver1</PNDeviceName>
                    </PROFINETDeviceName>
                </EthernetAddresses>
                <AdvancedOptions>
                    <RealTimeSettings>
                        <IOCommunication SendClock="1" />
                    </RealTimeSettings>
                </AdvancedOptions>
            </CentralDeviceInterface>
        </CentralDevice>
        <DecentralDevice DeviceRefID="V90_1">
            <DecentralDeviceInterface InterfaceRefID="V90_1_Interface">
                <EthernetAddresses>
                    <IPProtocol>
                        <SetInTheProject IPAddress="100.168.0.2"
                                         SubnetMask="255.255.255.0"/>
                    </IPProtocol>
                    <PROFINETDeviceName DeviceNumber="1">
                        <PNDeviceName>sinamics-v90-pn</PNDeviceName>
                    </PROFINETDeviceName>
                </EthernetAddresses>
            </DecentralDeviceInterface>
            <!-- GSDML reference of the plugged module: -->
            <Module ModuleID="DRIVE_OBJECT"
                    SlotNumber="1"
                    GSDRefID="IDM_DRIVE">
                <Submodule SubmoduleID="Module_Access_Point"
                           SubslotNumber="1"
                           GSDRefID="IDS_MAP">
                </Submodule>
                <Submodule SubmoduleID="without_PROFIsafe"
                           SubslotNumber="2"
                           GSDRefID="IDS_NOSAFE">
                </Submodule>
                <Submodule SubmoduleID="SIEMENS_telegram_111_PZD_12_12"
                           SubslotNumber="3"
                           GSDRefID="IDS_TEL111">
                    <IOAddresses>
                        <InputAddresses StartAddress="100" />
                        <OutputAddresses StartAddress="100" />
                    </IOAddresses> 
                </Submodule>
                <Submodule SubmoduleID="Supplementary_telegram_750_PZD_3_1"
                           SubslotNumber="4"
                           GSDRefID="IDS_TEL750">
                    <IOAddresses>
                        <InputAddresses StartAddress="200" />
                        <OutputAddresses StartAddress="200" />
                    </IOAddresses> 
                </Submodule>
            </Module>
        </DecentralDevice>
    </Devices>
</Configuration>
```
这个文件的插槽GSDRefID需要去GSDML里面查，排序可以参考codesys或博图，GSDML里也有允许顺序。之后，还需要修改**ListOfNodes.xml**，如下：
```
<ListOfNodes schemaVersion="1.0"
             ListOfNodesID="ListOfNodesID"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns="http://www.siemens.com/Automation/PNConfigLib/ListOfNodes"
             xsi:schemaLocation="http://www.siemens.com/Automation/PNConfigLib/ListOfNodes ListOfNodes.xsd">
    <PNDriver DeviceID="PN_Driver_1"
			  DeviceVersion="v3.1"
              DeviceName="PROFINET Driver">
        <Interface InterfaceID="PN_Driver_1_Interface"
                   InterfaceType="Linux Native"
                   InterfaceName="PN_Driver_1_Interface" />
    </PNDriver>
    <DecentralDevice DeviceID="V90_1"
                     GSDPath="../GSDMLs/GSDML-V2.32-Siemens-Sinamics_V90-20190415.xml"
                     GSDRefID="IDD_V90PN-V4.7"
                     DeviceName="V90_1">
        <Interface InterfaceID="V90_1_Interface"
                   InterfaceName="V90_1_Interface" />
    </DecentralDevice>
</ListOfNodes>
```

这样，再通过PNConfigLibRunner生成出来PROFINET Driver_PN_Driver_1.xml，我们就可以拿到PNDriver中使用了。

## 11.5 PNDriver的修改

我们测试只要跑通总线即可，这样只需要将中间XML拷到`examples/ioc/test_app/linux_native/PNDriverBase_TestApp/`，再修改pnd_test.c的init_paths部分即可。

需要注意的是vscode需要以管理员权限运行，通过`sudo code --no-sandbox --user-data-dir="./temp"`启动vscode，安装C++、CMAKE等插件就可以编译。

PNDriver不太容易看出来总线的状态和交互，可以结合WireShark抓包调试。通讯成功后，就可以把TestApp按需求重写。


## 11.6 结尾

少量的ProfiNET主站需求推荐用CODESYS/TwinCAT来做，有界面和诊断，调试起来很简单。PNDriver则适合于批量产品或特定软件研发，它还支持IRT同步（仅CP1625）和作为从站使用，具体参考示例代码。