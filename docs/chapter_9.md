# 9 附录：P-Net

## 9.1 P-Net简介

P-Net是rt-labs开发的一个ProfiNET从站协议栈，用C语言编写，实现了一致性A和B，RT，多网口等。  
在此之前，想要实现ProfiNET从站必须使用CODESYS等付费协议栈或者NetX等专用芯片。

WARNING: **开源协议**
P-Net是双协议开源，即对于开源爱好者采用GPLv3，而商业使用需要购买商业许可证。  
截至编写本文时，商业许可证价格最低为3000€报名费+490€年费+10€设备费（有阶梯价），例如2年生产1000台设备，费用大概为11200€，折合人民币不到9万，平均89/台设备。

官方的开源存储库在 [rtlabs-com/p-net](https://github.com/rtlabs-com/p-net) ，默认分支是public，主要是删除了demo程序。之前的代码可在master分支里看到。  
p-net有一个第三方衍生的c++版本叫做 [profipp](https://github.com/langmo/profipp) ，c++编程、自动生成GSDML。不过缺少文档，注释也少，有点看不懂。  
还有个三方移植在H723ZG上的，叫做 [p-net-stm32](https://github.com/andre-lost-a-pig/p-net-stm32) 。淘宝闲鱼有人卖基于stm32+w5500+交换机的，但固件不开源。  

本文主要讨论官方p-net在linux平台下的应用。  

## 9.2 确定需求

P-Net支持的功能很丰富，但我们的需求很简单，只需要和西门子建立通讯、周期稳定即可。要求主要是如下几点：  

- 可从博图上设置名称和ip
- 实时级别RT，单网口，周期4ms  
- InOut 64 Byte，类似codesys做从站
- 可扩展模块，为将来升级做准备

确定好需求后，我们开始看p-net内置的sample，并尝试跑起来。

## 9.3 平台搭建

系统建议debian12，打preempt补丁。可以参考 [LinuxCNC](https://linuxcnc.org/) 。  
安装依赖项：`sudo apt install cmake git`  
IDE推荐QT：`sudo apt install qt6-base-dev qtcreator`  

apt源里的qtcreator版本一般都比较老，要新版的可以到 [qt-creator/releases](https://github.com/qt-creator/qt-creator/releases) 下载最新版，不过相关依赖例如gdb、libxcb等需要手动安装。还需要注意的是虚拟机里可能会有问题，建议在物理机上跑。  

## 9.4 Sample例程

在编译Sample例程之前，我们要处理一下git问题。以下源代码基于 [p-net/tree/master](https://github.com/rtlabs-com/p-net/tree/master) 。  
P-Net依赖很少，但osal和googletest在国内环境不太好git。我们需要对原始cmake文件做一点小小的修改。  

1. 从 [p-net/tree/master](https://github.com/rtlabs-com/p-net/tree/master) 下载P-Net主体源代码，解压到设备中，目录结构**~/profinet/p-net**。  
2. 从 [rtlabs-com/osal](https://github.com/rtlabs-com/osal) 下载osal源代码，解压到p-net源码根文件夹中，目录名osal（~/profinet/p-net/osal）。  
2. 编辑CMakeLists.txt，屏蔽46行的`#include(AddOsal)`，并在228行添加`add_subdirectory (osal)`

这样就可以跳过osal的git。然后我们正常build：

```
cd ./profinet
cmake -B build -S p-net -DBUILD_TESTING=OFF -DUSE_SCHED_FIFO=ON
```

DBUILD_TESTING=OFF可以跳过gtest依赖，我们作为使用者也不需要测试环境。FIFO调度在rt系统上有助于增加实时性。  
编译完成后，我们可以找到~/profinet/build文件夹，里面就是编译后的二进制文件了。  
可以`sudo ~/profinet/build/pn_dev -h`打印信息。通常使用`sudo ~/profinet/build/pn_dev -i eth0`来指定网口运行。  
例程的GSDML文件在`~/profinet/p-net/samples/pn_dev`下。  

INFO: **混杂和多播**
P-Net在linux平台上也使用SOCK_RAW，具体代码在 `src/ports/linux/pnal_eth.c` 下，默认启用了多播`IFF_ALLMULTI`但未启用混杂`IFF_PROMISC` 。  
根据实际调试情况，某些系统、驱动可能需要启用混杂模式才可以正常接收PNIO_PS帧。

## 9.5 例程之上的修改

作为应用工程师，我们尽量在已有代码上做修改。P-Net模块很适合作为一个独立的后台程序运行，对外使用ProfiNET通讯，对内使用SharedMemory通讯。我们采用和CODESYS一致的POSIX方法，具体可以参考4.8章节的共享内存示例代码，数据的处理在`sampleapp_common.c`的`app_cyclic_data_callback`中，按实际需求编写用户代码。  

在正常通过终端编译之后，用QtCreator加载CMakeLists.txt，并将现有build文件夹作为生成目录，勾选Run as root user。后面就可以用QtCreator开发。  
GSDML只能手动修改，不过不复杂，主要是改Vendor、Device信息和插槽，这里举例把0x130作为8个usint输入：
```
         <ModuleList>
            <ModuleItem ID="IDM_30" ModuleIdentNumber="0x00000030">
               <ModuleInfo>
                  <Name TextId="TOK_Name_Module_I8"/>
                  <InfoText TextId="TOK_InfoText_Module_I8"/>
                  <HardwareRelease Value="1.0"/>
                  <SoftwareRelease Value="1.0"/>
               </ModuleInfo>
               <VirtualSubmoduleList>
                  <VirtualSubmoduleItem ID="IDSM_130" SubmoduleIdentNumber="0x0130" MayIssueProcessAlarm="true">
                     <IOData>
                        <Input Consistency="All items consistency">
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_1" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_2" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_3" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_4" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_5" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_6" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_7" UseAsBits="false" />
                           <DataItem DataType="Unsigned8" TextId="TOK_Input_DataItem_8" UseAsBits="false" />
                        </Input>
                     </IOData>
                     <ModuleInfo>
                        <Name TextId="TOK_Name_Module_I8"/>
                        <InfoText TextId="TOK_InfoText_Module_I8"/>
                     </ModuleInfo>
                  </VirtualSubmoduleItem>
               </VirtualSubmoduleList>
            </ModuleItem>
```

记得要同步修改`app_gsdml.h`中的信息和size，以及`sampleapp_common.c`中的数据处理。

## 9.6 结尾

有点可惜的是P-Net还不是真正的开源，收费模式和QT类似。现有demo的功能还是稍微复杂了点，能像igh那样作为独立服务就好了。
