# 8 附录：博图Openness

## 8.1 简介

西门子的产品给人印象最深刻的就是稳定性，但软件十分臃肿，内存占用高，编译和下载速度非常慢。开发环境需要授权，大多数开发人员都使用破解版，西门子也不对此做追溯限制。WinCC相似，不过终端客户会要求正版。

博图不必过多介绍，作为市场占有率最高的软硬件综合厂家，很容易找到书和文档，包括Openness。但是，西门子喜欢搞一些体量庞大但实用程度不高的东西，我认为Openness也在其中。网上现有的资料，尤其是官方示例和文档有些臃肿，撰写本文时基础文档就有1900页，还没有一个像样的Getting started和Demo code。当然，我在这里也不是写范例，更多的是想提供个参考或者思路。

Openness在非标自动化行业可以见到，主要原因是开发周期短、公司要求使用统一的模板。一个产线可能有数十个工站，工站有大量重复的内容，只是改一个数组序号。除此之外，气缸、传感器、伺服等执行器件需要做PlcTag、功能块调用、背景DB、组态屏和连接、显示文本等，每个项目、每个工站甚至详细到每个编号都手动输入是非常枯燥的。

Openness开放的是DLL接口，建议用C#编写程序。如果对C#开发不熟悉或者您只是PLC工程师，可以尝试用 [**Add-In中的Export-Import插件**](https://support.industry.siemens.com/cs/document/109773999){target=_blank}  来进行导出，导出的xml可以用Python做处理，处理完了仍然用Export-Import插件导入到项目中。不过这种方式容易导致博图崩溃或无法一次性导入所有内容。

## 8.2 基础需求

假设我们需要做一个产线，一共10个工位。我们的模板中只有一个工位和基础FB，需要以下几项：  

1. 公用的、全局的DB、FB，主体结构、数组和画面，这部分属于模板固有内容，我们不对它做变更。
2. 顺序控制相关代码，手自动、单步跳步、强制模式等功能，像是Main程序，我们对它进行部分变更，例如根据工位数量生成调用结构。
3. EPlan给出来的tag list，用于生成具体IO点位信息。tag中需要有一些标志，比如OP10_CY01_Home表示10站第一个气缸的原点。表格建议由人工二次标注，将气缸、传感器等组件用标识符分割及添加注释，方便程序筛选。
4. 模块的模板和单条内容xml，模板中包含头尾信息，单条内容包含IO关联、FB调用。

确定并编写基础模板的同时，我们也要确定自动生成要做哪些事情，主要包含以下几项：

1. 从eplan taglist中生成plc变量名和注释
2. 生成程序文件和文件夹结构、背景DB、结构体
3. 生成程序内容，主要包含气缸、传感器等执行器件的关联和调用
4. 根据生成的文件树，在MAIN程序中依次调用
5. 根据需求生成HMI结构、文本、报警列表等

除了用于PLC代码生成外，我们还可以用Openness来做库管理，可以根据项目需要用到的外设分配FB，减少资料泄露风险。

## 8.3 模板准备

模板准备需要用上面提到的Export-Import插件，将现有的程序导出。程序导出的是XML文件，语法是SimaticML。PLC代码建议全用SCL实现，XML还能有些可读性。导出后的文件主要是以下几类：

1. 程序、FB、背景DB都是SimaticML语法的XML文件。
2. 变量表、TextList属于比较特殊的XML，Textlist也可以以XLSX格式导入导出。
3. 文本列表在博图无插件上只能用XLSX，在Openness中只能用XML。

导出的模板文件需按照实际PLC的文件树分文件夹存储，方便之后按树形结构导入，也方便生成过程。  

关于模板XML的处理，我倾向于添加标识符，并在拷贝时识别标识符并替换。  
例如`DB_Sensor.OP[C_OP10].bSignal_1 := "OP10_Sensor1"`的SimaticML是以下这么一长串：

```
  <NewLine UId="279" />
  <Blank Num="8" UId="280" />
  <Access Scope="GlobalVariable" UId="281">
    <Symbol UId="282">
      <Component Name="DB_Sensor" UId="283">
        <BooleanAttribute Name="HasQuotes" UId="284">true</BooleanAttribute>
      </Component>
      <Token Text="." UId="285" />
      <Component Name="OP" UId="286">
        <Token Text="[" UId="287" />
        <Access Scope="GlobalConstant" UId="288">
          <Constant UId="289">
            <ConstantValue UId="290">C_OP10</ConstantValue>
          </Constant>
        </Access>
        <Token Text="]" UId="291" />
      </Component>
      <Token Text="." UId="292" />
      <Component Name="bSignal_1" UId="293" />
    </Symbol>
  </Access>
  <Blank UId="294" />
  <Token Text=":=" UId="295" />
  <Blank UId="296" />
  <Access Scope="GlobalVariable" UId="297">
    <Symbol UId="298">
      <Component Name="OP10_Sensor1" UId="299" />
    </Symbol>
  </Access>
  <Token Text=";" UId="300" />
```

这种单个条目还是比较短的，可以在C#中写死并对内容做替换。对于带IO关联、FB调用和REGION的代码，建议将其独立出来一个xml，C#程序中用StreamReader读取模板文件，用StreamWriter写目标文件/段落。这里不太推荐XmlWriter，层级多了不太好把控。  

## 8.4 表格读取

plc tag格如果是csv的，可以直接用StreamReader读取再由分隔符分割。如果是xlsx，可以用OpenXML。不过，OpenXML读写效率不高，相对于Python的pandas复杂不少。此处例举一个OpenXML读取过程：

```CSharp
SpreadsheetDocument spreadsheetDocument = SpreadsheetDocument.Open(ConfigurationFileLocation, false);
WorkbookPart workbookPart = spreadsheetDocument.WorkbookPart;
Sheet theSheet = workbookPart.Workbook.Descendants<Sheet>().FirstOrDefault(s => s.Name == "Sheet1");
WorksheetPart worksheetPart = (WorksheetPart)(workbookPart.GetPartById(theSheet.Id));
OpenXmlReader reader = OpenXmlReader.Create(worksheetPart);
while (reader.Read())
{
	if (reader.ElementType == typeof(Row))
	{
		reader.ReadFirstChild();
		do
		{
			if (reader.ElementType == typeof(Cell))
			{
				Cell c = (Cell)reader.LoadCurrentElement();
				string cellValue = "";
				if (c.DataType != null && c.DataType == CellValues.SharedString)
				{
					SharedStringItem ssi = workbookPart.SharedStringTablePart.SharedStringTable.Elements<SharedStringItem>().ElementAt(int.Parse(c.CellValue.InnerText));
					cellValue = ssi.InnerText;
				}
				else if (c.CellValue != null)
				{
					cellValue = c.CellValue.InnerText;
				}
				// Assemble cellValue to your structure.
			}
		}
	}
}

```


## 8.4 程序生成

SimaticML稍微有点绕，但不是很复杂。结构体、文本、程序和变量表的xml稍微有些区别，可以导出后查阅现有结构，根据现有结构生成所需要的程序。架构PLC模板时，主体结构尽量不动，只修改OPxxx的工站标号，导入时只需要对文件、文件夹和内容替换OP号。

需要自动生成的外设尽量放在一起，文件越少导入速度越快。对于中小型项目，也可以在模板中规划好OP10~OP200这样的目录结构，然后根据项目需求删除不用的目录，可以极大程度节省导入时间。

## 8.5 Openness操作

Openness整个框架还是比较复杂的，我们用到的主要是**导入**各类文件。

### 动态加载

官方推荐对DLL做动态加载，先从注册表中获取当前可用的版本：
```CSharp
RegistryKey baseKey = RegistryKey.OpenBaseKey(RegistryHive.LocalMachine, RegistryView.Registry64);
RegistryKey key = baseKey.OpenSubKey(@"SOFTWARE\Siemens\Automation\Openness\");
baseKey.Dispose();
if (key != null)
	RegistryKey key2 = key.OpenSubKey(0);
	key2 = key2.OpenSubKey("PublicAPI\\");
	if (key2 != null)
	{
		key2 = key2.OpenSubKey(names2.Last());
		EngineeringVersion = key2.GetValue("EngineeringVersion").ToString();
		EngineeringDllPath = key2.GetValue("Siemens.Engineering").ToString();
	}
}
```
在主程序中加载DLL解析器 `AppDomain.CurrentDomain.AssemblyResolve += CurrentDomain_AssemblyResolve;` ，解析以下几个DLL：

1. Siemens.Engineering.Contract.dll
2. Siemens.Engineering.ClientAdapter.Interfaces.dll
3. Siemens.Engineering.ClientAdapter.MarshallerHook.dll
4. Siemens.Engineering.ClientAdapter.MarshallerHook.Hmi.dll
5. Siemens.Engineering.dll

此处给出一个解析方式，其它DLL类似，不同版本需求可能不同，debug时会报错。DLL路径可以在博图安装目录里搜索。
```CSharp
private static Assembly CurrentDomain_AssemblyResolve(object sender, ResolveEventArgs args)
{
    int index = args.Name.IndexOf(',');
    if (index == -1)
        return null;

    string name = args.Name.Substring(0, index);

    if (name == "Siemens.Engineering.Contract")
    {
        string fullPath = string.Format("C:\\Program Files\\Siemens\\Automation\\Portal {0}\\Bin\\PublicAPI\\Siemens.Engineering.Contract.dll", tiaVersion);
        if (File.Exists(fullPath))
            return Assembly.LoadFrom(fullPath);
    }
}
```

虽然DLL是动态加载的，我们仍然要在VS中把Siemens.Engineering和Siemens.Engineering.Hmi拖到引用里，然后把复制本地改False。不然写代码时没加载到IDE的也就没有相关链接。

### 加载进程

在运行的进程里搜索博图实例，即打开一个博图IDE显示一个。我们简化一点，只考虑已运行单实例并且打开模板文件的情况。
```CSharp
TiaPortal tiaPortal = null;
foreach (TiaPortalProcess tiaPortalProcess in TiaPortal.GetProcesses())
{
	TiaPortalProcess p = TiaPortal.GetProcess(tiaPortalProcess.Id);
	tiaPortal = p.Attach();
	if (tiaPortal != null)
	{
		return;
	}
}

ProjectComposition projects = tiaPortal.Projects;
if (projects.Count == 0)
{
	return;
}
project = tiaPortal.Projects[0];
```

### 加载设备树

设备树主要用于区分PLC、HMI和总线设备，我们在这里提取出PLC和HMI，只考虑单PLC和单HMI的情况。需要注意的是，PLC使用TypeIdentifier可以识别出CPU型号，而HMI的标识类似订货号，不够明确，这里推荐在模板中增加标识符的方式来识别。

```CSharp
PlcSoftware software = null;
HmiTarget hmiTarget = null;
foreach (Device device in project.Devices)
{
	if (device.TypeIdentifier == null)
	{
		continue;
	}
	if (device.TypeIdentifier == "System:Device.S71500")
	{
		foreach (DeviceItem item in device.DeviceItems)
		{
			if (item.Classification.ToString() == "CPU")
			{
				//load software
				SoftwareContainer softwareContainer = ((IEngineeringServiceProvider)item).GetService<SoftwareContainer>();
				if (softwareContainer != null)
				{
					software = softwareContainer.Software as PlcSoftware;
				}
			}
		}
	}else if (device.DeviceItems.Count > 0)
	{
		//HMI not have Classification, only use name to check it.
		foreach (DeviceItem item in device.DeviceItems)
		{
			if (item.Name.ToString().Contains("HMI_TP"))
			{
				//load software
				SoftwareContainer softwareContainer = ((IEngineeringServiceProvider)item).GetService<SoftwareContainer>();
				if (softwareContainer != null)
				{
					hmiTarget = softwareContainer.Software as HmiTarget;
					hmiTargets.Add(hmiTarget);
				}
			}
		}
	}
}
```

### 导入过程

导入过程参考 [TiaOpennessHelper/Improt.cs](https://github.com/BrunoSantos-Git/TiaPortalOpenness/blob/master/Visual%20Studio/TiaOpennessHelper/Import.cs) ，TiaOpennessHelper里还有很多功能可以参考。

如果先导入了一个程序，但其引用的DB还未导入则会报错。可以加入`SWImportOptions.IgnoreMissingReferencedObjects`标识符以忽略错误，例如：
```CSharp
if (Path.GetExtension(filePath).Equals(".xml"))
	(destination as PlcBlockGroup).Blocks.Import(fileInfo, importOption, SWImportOptions.IgnoreMissingReferencedObjects);
```

## 8.6 结尾

Openness带Open但还不是非常开放，有不少元素修改不了或者需要花费精力研究博图底层架构，目前阶段建议只作为小工具替代繁琐的复制粘贴改序号。

程序员常用的git、copilot等工具在自动化行业还没有推开，甚至代码格式化在很多ide上都不完善。期待未来可以用ai集成编程，自然语言对话的辅助编程才是自动化调试的最终形态。

