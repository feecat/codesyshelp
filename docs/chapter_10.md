# 10 附录：IgH与Ruckig

## 10.1 方向选择

PLC封装了总线和功能，这带来了很大的便利性，我们只需要添加设备、调用功能、编写代码。但每个功能其实都是有偿的，包含在了硬件或授权费用中。非标行业由项目导向，每个项目差异不小，项目总金额大，用PLC可以方便多人开发多个站点。标准设备则可以编写C++代码替代PLC的功能，这是免费的，尤其适用于批量生产的、逻辑不复杂的标准品。

工业控制一般会在C#、C++或LabView中选一个，取决于项目需求，如果需要实时性或EtherCAT用C++，专职labview工程师选LabView，否则建议C#。本篇主要讨论C++、QT、IgH、Ruckig的单轴运动控制应用。 

## 10.2 平台搭建

Igh只支持Linux平台，建议使用 [LinuxCNC](https://linuxcnc.org/downloads/) 镜像，支持x86和树莓派平台，建议选PreemptRT版本。不推荐其它arm平台，树莓派已经是可用范围内的最低性能，RK和全志大部分性能不够或内核支持不好。

QT建议用 [Offline Qt Downloads](https://www.qt.io/offline-installers) 。断网安装可规避账户登录。下载Qt Creator和Qt6 source packages即可，从apt安装亦可，此处不列举具体安装过程。

IgH的源代码在 [EtherCAT Master](https://gitlab.com/etherlab.org/ethercat) ，文档在 [ethercat_doc_1.6](https://docs.etherlab.org/ethercat/1.6/pdf/ethercat_doc.pdf) 。

[Ruckig](https://github.com/pantor/ruckig) 用于伺服轨迹规划，它其实主要用于多轴机器人运动规划，在单轴上打断运动可能会有问题，但可以通过代码规避。

## 10.3 IgH编译

撰写本文时igh版本为1.6。版本不同，编译和使用过程可能有些许不同。  
安装相关依赖：`sudo apt-get install git automake autoconf libtool pkg-config make build-essential net-tools` 。  
解压缩后，进入解压后的目录，开始编译。编译过程参考 [ros2_ethercat](https://github.com/ros2torial/ros2_ethercat/blob/main/INSTALL_IGH_ETHERCAT_MASTER.md) ，有删改。
```bash
sudo ./bootstrap
sudo ./configure --prefix=/usr/local/etherlab  --disable-8139too --enable-generic --enable-r8169 --enable-igb --disable-eoe
sudo make all modules
sudo make modules_install install
sudo depmod
sudo ln -s /usr/local/etherlab/bin/ethercat /usr/bin/
sudo ln -s /usr/local/etherlab/etc/init.d/ethercat /etc/init.d/ethercat
sudo ldconfig /usr/local/etherlab/lib
sudo mkdir /etc/sysconfig
sudo cp /usr/local/etherlab/etc/sysconfig/ethercat /etc/sysconfig/ethercat
```
常用网卡是RTL8111和I210，分别对应r8169和igb驱动。如果不能用这俩，generic可以跑但实时性较差。  
安装完成后编辑`/etc/sysconfig/ethercat`配置mac地址和驱动，使用`sudo /etc/init.d/ethercat start`启动，`sudo ethercat master`查询状态。  

如果需要自动启动服务，则编辑`/usr/local/etherlab/etc/ethercat.conf`，配置文件修改后使用如下命令启动服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable ethercat.service
sudo systemctl start ethercat.service
```

## 10.4 获取通讯结构体

PLC使用设备描述文件，图形化地添加设备。部分商业EtherCAT主站也支持自动扫描、自动配置，但IgH尚不支持。  
EtherCAT服务启动后，连接上硬件，执行`sudo ethercat slaves`应当可以得到如下信息：
```
0  0:0  PREOP  +  SV660_1Axis_00916
```
具体的解释可以参考文档7.1.20章节。从站能进PREOP说明连接正常、服务正常，接下来获取通讯结构体，执行`sudo ethercat csruct`读取从站的PDO信息，这是默认的PDO，对应CODESYS、Twincat插入设备后默认的pdo组，可以修改但要参考XML。这里我们不修改，存起来直接使用。
```cpp
/* Master 0, Slave 0, "InoSV660N"
 * Vendor ID:       0x00100000
 * Product code:    0x000c010d
 * Revision number: 0x00010000
 */

ec_pdo_entry_info_t slave_0_pdo_entries[] = {
    {0x6040, 0x00, 16},
    {0x607a, 0x00, 32},
    {0x60b8, 0x00, 16},
    {0x60fe, 0x01, 32},
    {0x603f, 0x00, 16},
    {0x6041, 0x00, 16},
    {0x6064, 0x00, 32},
    {0x6077, 0x00, 16},
    {0x60f4, 0x00, 32},
    {0x60b9, 0x00, 16},
    {0x60ba, 0x00, 32},
    {0x60bc, 0x00, 32},
    {0x60fd, 0x00, 32},
};

ec_pdo_info_t slave_0_pdos[] = {
    {0x1600, 4, slave_0_pdo_entries + 0},
    {0x1a00, 9, slave_0_pdo_entries + 4},
};

ec_sync_info_t slave_0_syncs[] = {
    {0, EC_DIR_OUTPUT, 0, NULL, EC_WD_DISABLE},
    {1, EC_DIR_INPUT, 0, NULL, EC_WD_DISABLE},
    {2, EC_DIR_OUTPUT, 1, slave_0_pdos + 0, EC_WD_ENABLE},
    {3, EC_DIR_INPUT, 1, slave_0_pdos + 1, EC_WD_DISABLE},
    {0xff}
};

```

## 10.5 创建QT项目

网上有很多QT、C++的基础文档，这里不过多介绍。创建一个Qt Widgets Application，pro文件中添加以下两行来链接igh库：
```cpp
INCLUDEPATH += /usr/local/etherlab/include
LIBS += -L/usr/local/etherlab/lib -lethercat
```
大多数应用中，EtherCAT都是独立出一个线程以保证实时性的。在main中创建线程：
```cpp
    QThread *threadEC = QThread::create(ecMain);
    threadEC->start();
```

ecMain的代码参考官方`examples/dc_user/main.c`，主要修改以下几处：

1. 添加`#define driverPos 0,0`和`#define driverVendor 0x00100000, 0x000c010d`。vendor信息在上面cstruct里。
2. 将cstruct完整复制到源代码中，有的伺服初始状态不一定正确，可以先用CODESYS连一下，伺服不断电再用igh获取cstruct。
3. 创建offsetPdo和domain registers。这部分参考 [is620n_dc.c](https://github.com/liuzq71/igh-Master-samples/blob/9e90a41911f9e434ae7a9a8f4640a8949fd9d6d3/is620n_dc.c#L106-L157)。只参考这部分，其它还是用igh最新版示例代码。
4. 创建/修改初始化代码，也可以参考上面的is620n。DC设置推荐0x300模式，同步偏移推荐PERIOD_NS/2。
5. 在cyclic_task中读写pdo，并创建自己的process_app，举例如下：

```cpp
	drv.status_word = EC_READ_U16(domain1_pd + offsetPdoSV660N.status_word);
	drv.position_actual_value = EC_READ_S32(domain1_pd + offsetPdoSV660N.position_actual_value);
	process_app();
	EC_WRITE_U16(domain1_pd + offsetPdoSV660N.control_word, drv.control_word);
	EC_WRITE_S32(domain1_pd + offsetPdoSV660N.target_position, drv.target_position);
```

igh默认的断线重连机制有bug，尤其是在程序中断时（程序中断会导致ethercat通讯中断）可能卡在SAFEOP状态。可通过` ecrt_master_state(master, &master_state);`获取状态，如果al_states超过10秒到不了8，可通过`ecrt_master_reset(master);`复位主站。

## 10.6 CiA402使能

绝大多数EtherCAT伺服都是CiA402标准的，上使能无非reset、enable voltage、switch on、enable operation几个阶段，可以参考 [OpenSML_Power](https://github.com/feecat/OpenSML/blob/master/src/OpenSML_Power.txt)  ，C++其实也是拉一个case分步，之后对控制字、状态字进行操作。  
C++的读位比较简单，如下：
```cpp
bool aa = bb & (1 << 2);//读取bb的第二位（从右往左，下同）
```
写位要区分是置1还是赋值，如果是置1，如下（适用于上使能时的仅置位）：
```cpp
aa |= 1 << 2;//置位变量aa的第二位
```
如果是位赋值，建议独立出来（适用于Home等带置位复位的操作）：
```
void setBit(uint16_t *data, uint8_t bit, uint8_t val){
    if (val)
        *data |= 1 << bit;
    else
        *data &= ~(1 << bit);
}
```
由于我们使用CSP模式，需要在上使能的过程中将position actual value赋给target position，如果上完使能后位置差异过大会飞车。在调试过程中，一定要注意target position不能随意赋值，并在调试时脱开机械连接。

## 10.7 Ruckig

PLC做运动控制的一大特点就是可以实施jerk限制的S型速度曲线，绝大多数运动控制都需要。计算一个曲线不复杂，打断任意状态的运动并平滑速度的过渡是很困难的，各大厂商也都不开放这部分代码。Ruckig实现了大部分功能，可以用于实际项目中。感兴趣可以参考以下几个链接：

1. [Jerk-limited Real-time Trajectory Generation with Arbitrary Target States](https://arxiv.org/pdf/2105.04830)
2. [Comparison of Different Motion Types](http://support.motioneng.com/Software-MPI_04_00/Topics/diff_mtn_types.htm)
3. [Mathematics of Motion Control Profiles](https://www.pmdcorp.com/resources/type/articles/get/mathematics-of-motion-control-profiles-article)
4. [A Primer on Bézier Curves](https://pomax.github.io/bezierinfo/)
5. [Curves and Surfaces](https://ciechanow.ski/curves-and-surfaces/)

Ruckig官方提供了示例 [01_position.cpp](https://github.com/pantor/ruckig/blob/main/examples/01_position.cpp) ，把轴数量改为1，并把while删掉即可。如下：

```
ruckig::Ruckig<1> otg(0.001);
ruckig::InputParameter<1> input;
ruckig::OutputParameter<1> output;

// Set input parameters
input.current_position = {axis.targetPosition};//use last cycle position
input.current_velocity = {axis.targetVelocity};
input.current_acceleration = {axis.targetAcceleration};

input.target_position = {axis.nextPosition};//this is real of target position
input.target_velocity = {0.0};//end velocity always as 0
input.target_acceleration = {0.0};//end acceleration always as 0

input.max_velocity = {axis.max_velocity};
input.max_acceleration = {axis.max_acceleration};
input.max_jerk = {axis.max_jerk};

if (otg.update(input, output) >= 0) {
    output.pass_to_input(input);
    axis.targetPosition = output.new_position[0];
    axis.targetVelocity = output.new_velocity[0];
    axis.targetAcceleration = output.new_acceleration[0];
}
```

这是比较浪费性能的做法，因为每个周期都会进行完整计算。也可以参考 [02_position_offline.cpp](https://github.com/pantor/ruckig/blob/main/examples/02_position_offline.cpp) 离线计算。实际应用中，还需要对targetPosition进行缩放，以适应实际的执行端单位。缩放需考虑越界，可以参考4.8章检测最高两位，移植到C++平台即可。还需要注意的是，x86和arm平台有不同的转换逻辑，double/float建议先转int再转uint以避免无法给定负值。

在上面的示例中，我们使用max_velocity作为运动过程的最大速度，而ruckig标准逻辑中max_velocity应当是不变的限制值，如果超出了限制值，ruckig会将运动拉到一个可计算的程度，这会造成不确定的运动如过冲、震荡或逆向运动。这在 [#45](https://github.com/pantor/ruckig/issues/45) 、 [#17](https://github.com/pantor/ruckig/issues/17) 等都有提到。一个粗劣的解决方法是，打断时检测最大速度是否发生了降低且降低超过一定值，如果发生了，则将ruckig模式切换为速度模式，待速度执行完后再切换回位置模式。这又带来了一个新的问题，如果轨迹的距离过短，这会造成比正常计算更多的过冲量。  
如果在打断运动时不降低最大速度，或满足一些条件如在减速阶段降低最大速度，则没有这个问题。

## 10.8 整合

代码需要把各个功能整合在一起，即最核心的是ruckig轨迹计算，外面一层是softmotion相关功能，封装为process_app。再外面依次是ethercat、线程和QT框架。实际应用中，还可以把运动控制部分和上层UI分开，以避免调试时的中断打断ethercat通讯。

有些情况下也可以用伺服的PP、PV模式，这样不需要ruckig，对实时性的要求也更低。但不适用于频繁打断运动或跑复杂运动控制的场景。

虽然windows也有实时应用的案例，但实现起来都比linux复杂很多，绝大多数还需要授权费。非实时应用或纯采集、分析的应用可以选c#，开发更快，windows平台也更容易维护。



