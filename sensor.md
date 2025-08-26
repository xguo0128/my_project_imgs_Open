---
title: 'Hi3093开发板传感器功能适配与调试'
date: '2025/08/25'
tags:
  - 传感器配置
  - I2C工具
sig: sig-bmc-core
archives: '2025-08'
author:
  - name: '郭馨'
    title: 'Hi3093开发板传感器功能适配与调试'
    description: 'description'
    avatar: 'avatar'
description: "Just about everything you'll need to style in the theme：headings, paragraphs, blockquotes, tables, code blocks, and more."
---
## 前言
> 本文介绍了基于[米尔电子MYC-LHi3093开发板](https://www.myir.cn/shows/136/71.html)适配i2c传感器的常用调试和适配方法，供社区开发者适配传感器提供指导。

## 物料准备
- Hi3093核心板
- 核心板电源适配器
- 网线
- typeC-USB线
- 杜邦线
- 扩展板（拥有LM75传感器）

## I2C工具的应用
目前2506版本的OpenUBMC暂不支持i2ctool工具进行i2cdetect和i2ctransfer操作，本文参考论坛帖子[自制I2C调试工具分享(适用于OpenUBMC平台的i2cdetect和i2ctransfer)](https://discuss.openubmc.cn/t/topic/2031)，将i2cdetect和i2ctransfer工具通过scp上传到/data目录下使用。

## 传感器的调试
根据米尔科技开发板硬件原理图设计，通过开发板上的排针将BMC的I2C总线等通过杜邦线连接到外部调试板上。本文以I2C9为例，电源、GND、SDA和SCL连线如下图所示：

![Hi3093开发板](https://cdn.jsdelivr.net/gh/xguo0128/my_project_imgs_Open/Hi3093_dev_board.jpg)

已知连接到外部调试板上的I2C9下挂载有LM75(0x48)、LM75(0x4f)、PCA9546(0x70)，PCA9546的channel0下接有LM75(0x49)。
通过/data目录下执行./i2cdetect 9可看出上述设备已经正确探测到。

![i2c检测结果](https://cdn.jsdelivr.net/gh/xguo0128/my_project_imgs_Open/i2cdetect9.png)

通过执行./i2ctransfer 9 w2@0x70 0x00 0x01，使能PCA9546 channel0，再执行./i2cdetect 9可看出LM75(0x49)可以被探测到。

![i2c探测结果](https://cdn.jsdelivr.net/gh/xguo0128/my_project_imgs_Open/pca9548_channel0.png)

## 传感器的配置
CSR配置以board name为openUBMC为例，vpd/vendor/Huawei/Server/Kunpeng/openUBMC/root.sr 中配置如下：
- "ManagementTopology"-> "Anchor"->"Buses"内增加 "I2c_9"
- "I2c_9"下增加LM75传感器芯片和"Connectors"配置， "Connector"下配置"Connector_DemoCard"，以表示PCA9546位于下级组件DemoCard的CSR。
- "Objects"下配置LM75对象、Entity、ThresholdSensor，因该传感器是定时轮询获取数值，故配置Scanner。
- "Objects"下配置"Connector_DemoCard"，"IdentifyMode"配置为2表示下一级组件非天池组件，非天池组件由Bom、Id、AuxId三项配置决定下一级组件CSR名称: Bom_Id_AuxId.csr，本例中下一级组件CSR名称为14100513_DemoCard_0.sr。"Buses"中配置的"I2c_9"表示和下一级板卡连接的总线为I2C9，需注意同一个CSR文件中的任意两个连接器Position不能重复。

```json
{
 "ManagementTopology": {
        "Anchor": {
            "Buses": [
                "I2c_1",
                "I2c_2",
                "I2c_3",
                "I2c_4",
                "I2c_5",
                "I2c_6",
                "I2c_7",
                "I2c_8",
                "I2c_9",
                "I2c_11"
            ]
      },
	 "I2c_9": {
            "Chips": [
                "Lm75_1",
                "Lm75_2"
            ],
            "Connectors": [
                "Connector_DemoCard"
            ]
       }
    },
	"Objects": {
	        "I2c_9": {
            "Id": 9
        },
        "Lm75_1": {
            "OffsetWidth": 1,
            "AddrWidth": 1,
            "Address": 144
        },
        "Lm75_2": {
            "OffsetWidth": 1,
            "AddrWidth": 1,
            "Address": 158
        },
        "Scanner_InletLM75": {
            "Chip": "#/Lm75_1",
            "Size": 1,
            "Offset": 0,
            "Mask": 255,
            "Period": 1000
        },
        "Scanner_OutletLM75": {
            "Chip": "#/Lm75_2",
            "Size": 1,
            "Offset": 0,
            "Mask": 255,
            "Period": 1000
        },
        "ThresholdSensor_InletTemp": {
            "AssertMask": 516,
            "DeassertMask": 516,
            "ReadingMask": 4626,
            "Linearization": 0,
            "M": 100,
            "RBExp": 224,
            "UpperCritical": 60,
            "LowerCritical": 5,
            "PositiveHysteresis": 2,
            "NegativeHysteresis": 2,
            "OwnerId": 32,
            "OwnerLun": 0,
            "EntityId": "<=/Entity_InletTemp.Id",
            "EntityInstance": "<=/Entity_InletTemp.Instance",
            "Initialization": 127,
            "Capabilities": 104,
            "SensorType": 1,
            "ReadingType": 1,
            "SensorName": "Inlet_Temp",
            "Unit": 128,
            "BaseUnit": 1,
            "ModifierUnit": 0,
            "Analog": 1,
            "MaximumReading": 127,
            "MinimumReading": 128,
            "Reading": "<=/Scanner_InletLM75.Value"
        },
        "Entity_InletTemp": {
            "Id": 99,
            "Name": "Inlet_Temp",
            "PowerState": 1,
            "Presence": 1,
            "Instance": 96
        },
        "ThresholdSensor_OutletTemp": {
            "AssertMask": 29312,
            "DeassertMask": 29312,
            "ReadingMask": 6168,
            "Linearization": 0,
            "M": 100,
            "RBExp": 224,
            "UpperCritical": 65,
            "UpperNoncritical": 50,
            "PositiveHysteresis": 2,
            "NegativeHysteresis": 2,
            "OwnerId": 32,
            "OwnerLun": 0,
            "EntityId": "<=/Entity_OutletTemp.Id",
            "EntityInstance": "<=/Entity_OutletTemp.Instance",
            "Initialization": 127,
            "Capabilities": 104,
            "SensorType": 1,
            "ReadingType": 1,
            "SensorName": "Outlet_Temp",
            "Unit": 128,
            "BaseUnit": 1,
            "ModifierUnit": 0,
            "Analog": 1,
            "MaximumReading": 127,
            "MinimumReading": 128,
            "Reading": "<=/Scanner_OutletLM75.Value"
        },
        "Entity_OutletTemp": {
            "Id": 99,
            "Name": "OutletTemp",
            "PowerState": 1,
            "Presence": 1,
            "Instance": 97
        },
        "Connector_DemoCard": {
            "Bom": "14100513",
            "Slot": 2,
            "Position": 4,
            "Presence": 1,
            "Id": "DemoCard",
            "AuxId": "0",
            "Buses": [
                "I2c_9"
            ],
            "SystemId": 1,
            "ManagerId": "1",
            "ChassisId": "1",
            "IdentifyMode": 2
        }
    }
}
```
因新增了CSR文件，需要在vpd/vendor/Huawei/Server/Kunpeng/openUBMC/profile.txt中增加14100513_DemoCard_0.sr文件所在路径。本例中配置如下：
```
Huawei/Server/Kunpeng/openUBMC/PSR/14100513_DemoCard_0.sr
```
14100513_DemoCard_0.sr配置如下：
```json
{
    "FormatVersion": "3.00",
    "DataVersion": "3.00",
    "ManagementTopology": {
      "Anchor": {
            "Buses": [
                "I2c_9"
             ]
      },
      "I2c_9": {
        "Chips": [
          "Pca9545_i2c9_chip"
        ]
      },
      "Pca9545_i2c9_chip": {
        "Buses": [
          "I2cMux_Pca9545_i2c9_chan0"
        ]
      },
      "I2cMux_Pca9545_i2c9_chan0": {
        "Chips": [
              "Lm75_3"
        ]
      }
    },
    "Objects": {
        "Pca9545_i2c9_chip": {
            "OffsetWidth": 0,
            "AddrWidth": 1,
            "Address": 224,
            "WriteTmout": 0,
            "ReadTmout": 0,
            "HealthStatus": 0
        },
        "I2cMux_Pca9545_i2c9_chan0": {
            "ChannelId": 0
        },
        "Lm75_3": {
            "OffsetWidth": 1,
            "AddrWidth": 1,
            "Address": 146
        },
        "Scanner_NicLM75": {
            "Chip": "#/Lm75_3",
            "Size": 1,
            "Offset": 0,
            "Mask": 255,
            "Period": 1000
        },
        "ThresholdSensor_DemoTemp": {
            "AssertMask": 31381,
            "DeassertMask": 31381,
            "ReadingMask": 16191,
            "Linearization": 0,
            "M": 100,
            "RBExp": 224,
            "UpperNonrecoverable": 70,
            "UpperCritical": 65,
            "UpperNoncritical": 55,
            "LowerNonrecoverable": 0,
            "LowerCritical": 5,
            "LowerNoncritical": 10,
            "PositiveHysteresis": 2,
            "NegativeHysteresis": 2,
            "OwnerId": 32,
            "OwnerLun": 0,
            "EntityId": "<=/Entity_NicTemp.Id",
            "EntityInstance": "<=/Entity_NicTemp.Instance",
            "Initialization": 127,
            "Capabilities": 104,
            "SensorType": 1,
            "ReadingType": 1,
            "SensorName": "Nic_Temp",
            "Unit": 128,
            "BaseUnit": 1,
            "ModifierUnit": 0,
            "Analog": 1,
            "MaximumReading": 127,
            "MinimumReading": 128,
            "Reading": "<=/Scanner_NicLM75.Value"
        },
        "Entity_NicTemp": {
            "Id": 9,
            "Name": "Nic_Temp",
            "PowerState": 1,
            "Presence": 1,
            "Instance": 98
        }
    }
}
```
## 传感器Web显示
经过上述配置后，可看到传感器Web已显示正常。
![传感器Web显示](https://cdn.jsdelivr.net/gh/xguo0128/my_project_imgs_Open/传感器web显示.png)
