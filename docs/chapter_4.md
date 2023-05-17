# 4 visu、softmotion、附加功能

## 4.1 Visu结构组成

Visuallization字面意思即可视化，可以理解为内置的组态屏。最常用的是WebVisu，可以通过浏览器本地或局域网内访问。本地用chromium的kiosk模式打开即和TargetVisu效果一致，且配置更自由。故现在很少会用TargetVisu。

在做Visu前，建议各位读者先有一个设计思路。一般情况下，我们推荐元素不要用太多的种类，颜色可以丰富一些并用浅色，字体应用黑体或仿宋，背景可以用浅色纯色或白色90%透明度叠加的背景图片。通常情况下，我们不建议使用官方的指示灯、切花开关、码表等元素，因为图标风格不一致会有设计上的割裂感。按钮、指示灯、文字和框架都可以用矩形来做。Style选择Default或Flat Style。一般不建议在一个界面中挤满元素，而是通过分页、分框架的形式构建界面。

我们可以在Application下添加visu，添加第一个visu时会自动添加Visulization Manager。添加visu时会有提示是否勾选VisuSymbols，这是系统内置的一些SVG图片库，可根据需求勾选。

添加的Visu默认是一个Visualization，可以理解为一个页面。如果需要做对话框可以改变visu属性为dialog。在属性中也可以设置visu的页面大小，该大小为画布大小，还会在管理器里根据缩放选项缩放。

界面右侧会有Visualization Toolbox，用的最多的是Basic中的矩形、圆形、直线和Frame。

在制作背景、图片按钮或选框时，我们还要在项目中插入ImagePool和TextPool，它们用于给图片或文本分配ID，即可在Visu中静态或动态引用。
	
## 4.2 基础元素

直线：直线是最常用的用于分隔界面小块的元素，也有少部分作为旋转指针实现。当作为分割线时，建议在Apperance中设置Line width在2至4之间，线形可以根据需要选择，颜色建议浅灰色或深灰色。横竖的对齐需要手动编辑Position下的Points坐标实现。

矩形：矩形（或圆角矩形）是最常用的元素，它们既可以作为指示灯，又可以作为按钮，还可以显示文字、信息，亦可以在框架中作为下拉菜单选框使用。

当作为双色切换的指示灯时，编辑Colors下的Normal state和Alarm state下的Fill color，建议一个改为LightGray，另一个改为LightGreen。并将Apperance下的Line style改为Invisible（或Hollow）。再在Color variables的Toggle color关联变量即可。

当作为按钮时，可以在Texts中输入文字作为按钮标识，并在Input configuration中关联动作。若需要带指示的按钮，可重复指示灯操作。

当作为文本框时，只需要写入文字并将边框隐藏即可，可在Text properties中设置文本属性和对齐方式。

除了这两种基础元素外，常用的还有Image（插入图片），Frame（框架），Label（文本），Button（按钮），Group Box（组合框），Slider（滑动按钮），CheckBox（选择框）等。

## 4.3 界面框架

## 4.4 运动控制简介

运动控制（Softmotion）是codesys的一大特色，softmotion通过库中的plc逻辑计算每个周期轴应当处在的位置，并通过总线发送达到实时控制的目的，驱动器工作在CSP模式下。除了softmotion外，codesys还支持softmotion light，plc发送非循环的指令控制，驱动器工作在PP、PV等模式下。Softmotion light还可以通过OpenSML或自己写逻辑实现，不需要授权。

运动控制的授权会在各个授权中标明是否含有，例如树莓派的标准授权中不含运动控制，若需要使用轴或轴组功能则需要额外购买授权。但其可以通过自定义轴结构体并手动映射变量规避，需要一定的编程能力。

## 4.5 轴与轴组

## 4.6 凸轮与插补

## 4.7 配方和文件功能

## 4.8 日期、大小端、特殊功能

### 日期

Codesys默认是从系统读取时间，系统依赖于RTC时钟。由于各厂家RTC时钟设置有差异，我们不去讨论设置时间，而是看看如何读取时间。需要在库管理器中添加`SysTime`和`SysTypes2 Interfaces`库。
```iecst
VAR
	udiUtcTime:UDINT;
	Result:RTS_IEC_RESULT;
	udiUtcTimeLocal:UDINT;
	dtDateAndTime:DT;
	sDateAndTime:STRING;
	
	sDate:STRING;
	sTime:STRING;
END_VAR

udiUtcTime := SysTimeRtcGet(Result);//获取时间
Result := SysTimeRtcConvertUtcToLocal(udiUtcTime, udiUtcTimeLocal);//转换为UTC时间
dtDateAndTime := UDINT_TO_DT(udiUtcTimeLocal);//将时间从udint转换为DT
sDateAndTime := DT_TO_STRING(dtDateAndTime);//将时间从DT转换为STRING，将自动补0

sDate := MID(sDateAndTime , 10 , 4);//获取当前日期，格式2001-01-01
sTime := RIGHT(sDateAndTime , 8);//获取当前时间，格式00:00:00
```

### 大小端

在和西门子或Modbus设备通讯时容易遇到大小端问题，一个32位的数据会拆分成4个byte，如果发送方是ABCD，则接收端可能是CDAB，这就是大小端问题。
大小端有很多种解法，例如创建数组关联每一位、创建联合体等，在这里我提出一个较简单的写法：
```iecst
FUNCTION FUN_EC : DWORD
VAR_INPUT
	dwInput:DWORD;
END_VAR

FUN_EC:=SHL(dwInput,16) OR SHR(dwInput,16);
```
该方法只适用于DWORD，如果您使用的变量是DINT或REAL，则外部还需要转换，例如：
```iecst
aa := DWORD_TO_DINT(FUN_EC(bb));
cc := FUN_EC(DINT_TO_DWORD(dd));
```
	
### 关机

所有的带系统的PLC，如Linux、WES7、WIN10系统，都需要正常的关机流程或配备UPS模块。不带系统的或经过OEM厂商特殊优化的Linux除外。如果没有优化过系统断电，则需要手动关机。为了不破坏用户体验，一般会在界面上写一个按钮或关联UPS的输出变量来自动关机。
	

- Windows系统（ControlWin和ControlRTE一致）：在库管理器中添加`SysProcess`和`SysTypes2 Interfaces`库，在程序中使用：

```iecst
VAR
    xShutdown: BOOL;
    Result: RTS_IEC_RESULT;
END_VAR

IF xShutdown THEN
    SysProcess.SysProcessExecuteCommand('shutdown -s -t 0', ADR(Result));
    xShutdown := FALSE;
END_IF
```

- Linux系统：编辑配置文件，在`[SysProcess]`下改为**Command=AllowAll**，在库管理器中添加`SysProcess`和`SysTypes2 Interfaces`库，在程序中：
	
```iecst
VAR
    xShutdown: BOOL;
    Result: RTS_IEC_RESULT;
END_VAR

IF xShutdown THEN
	SysProcess.SysProcessExecuteCommand('sudo shutdown now', ADR(Result));
    xShutdown := FALSE;
END_IF
```

### 获取序列号

如果系统对客户开放，则第三方也可以拷出来编译后的二进制文件运行（但不可修改）。若有加强保密的需要，可添加序列号校验。该方式不是绝对的安全，编译后的二进制文件只有CRC校验，没有混淆加密，相对于运行时更易被破解。在库管理器中添加`SysTarget`和`SysTypes2 Interfaces`库，在程序中：

```iecst
VAR
    Result: RTS_IEC_RESULT;
	psSN: POINTER TO STRING(255);
	sSN : STRING(255);
	maxLen: DINT := 255;
END_VAR

IF psSN = 0 THEN
	Result := SysTarget.SysTargetGetSerialNumber(ADR(psSN),ADR(maxLen));
END_IF
sSN := psSN^;
```


### 处理变量越界

一个光栅式单圈编码器转一圈会回零，在驱动中会用已记录的圈数乘以一圈脉冲量再加上当前脉冲量来获取位置。变量也是类似，一个16位的变量会越界，越界结果是回零或正/负极限。根据突变规律可以将16位扩展到32位，就不会越界了。
```iecst
FUNCTION_BLOCK FB_PosCalc
VAR_INPUT
	In:DINT;
	xReset:BOOL;
END_VAR
VAR_OUTPUT
	Out:LINT;
END_VAR
VAR
	udiTemp:UDINT;
	byTemp:byte;
	byTempOld: BYTE;
	diTemp:DINT;
	udiOffset: UDINT;
END_VAR
```
功能块程序：
```iecst
//越界后正负不好判断，转为UDINT后判断越界
udiTemp:=DINT_TO_UDINT(In);

//正向越界（11xxx=>00xxx），当前值00，旧值11。反向越界（00xxx=>11xxx）,当前值11，旧值00。
byTemp.0:=udiTemp.30;
byTemp.1:=udiTemp.31;
IF byTemp = 0 AND byTempOld = 3 THEN
	diTemp:=diTemp+1;
ELSIF byTemp = 3 AND byTempOld = 0 THEN
	diTemp:=diTemp-1;
END_IF
byTempOld:=byTemp;

//高位移位，正负依然有效（最高位）。低位借用当前值减偏置值
Out:=diTemp;
Out:=ROL(Out,32);
Out:=Out+udiTemp-udiOffset;

IF xReset THEN
	udiOffset:=udiTemp;
	diTemp:=0;
END_IF
```
