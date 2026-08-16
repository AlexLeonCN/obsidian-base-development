# 一、背景

该项目用于进行车辆服务编排，可以创建工程、ECU、总线、报文、信号信息，支持数据导入导出
# 二、框架

后端服务位于项目根目录的`./serve`目录下

使用SpringBoot框架，集成mabatis框架来连接和操作数据库

数据库使用内嵌式数据库H2，采用本地文件的持久化存储模式

前端工程在项目根目录的`ui`目录下，采用VUE.JS, 并使用后端SpringBoot的内嵌式Tomcat来启动

# 三、规则
- 需要在项目根目录下创建一个build.sh脚本，该脚本可以将前端工程自动打包到后端服务的静态资源目录，同时可以将后端服务打包成jar
- 每个持久化对象都需要在数据库schema.sql文件中定义dll建表语句
- 每个持久化对象都要配套有mapper接口和mapper.xml来定义接口和sql语句
- 需要在项目的单元测试中创建一个initDatabase方法，该方法执行后可以清除并重新创建H2的.mv文件，并基于schema.sql重新创建数据库表格
- 所有主键ID通过雪花算法生成
- 所有MAC地址的输入，在前端都自动展示:冒号不用手动输入冒号,冒号之间为输入框分段输入，传递给后端的时候不用带上冒号
- 所有IP地址的输入，在前端都自动展示.点号不用手动输入，点号之间为输入框分段输入，传递给后端的时候需要带上点号的字符串
- 使用ErrorContant.java interface类型来定义错误码，内部按照业务划分子interface，子interface包含多个错误码键值对， 键值对采用Pair<Integer, String>, 含义分别是错误码和报错信息。 错误码为8位，前4位从1000开始按照业务场景递增， 后4位从0001开始按照错误递增
- Pair是自定义类，支持范型构建
- 需要自定义异常VCDPException，支持基于ErrorContant中定义的Pair去构建，也支持基于错误码和报错信息来构建，在业务流程中抛出VCDPException

# 四、开发流程
## 1. 项目工程

### 页面功能

进入项目首页后，展示项目列表

工程以卡片网格形式展现，需要展现的字段有：名称、描述

项目很多时要支持分页查询

卡片上要有编辑、删除功能

工程列表页面需要有新增、批量删除

### 数据定义

#### 1) Project

| 字段   | 类型     | 含义   |
| ---- | ------ | ---- |
| id   | String | 主键ID |
| name | String | 工程名称 |
| desc | String | 描述   |

对应前端表单：工程配置

| 名称   | 对应字段 | 是否必填 |
| ---- | ---- | ---- |
| 工程名称 | name | 是    |
| 描述   | desc | 否    |

校验规则：
- 工程名称`name`不可与数据库中存量工程重复

## 2. ECU配置

### 页面功能

点击进入工程后，左侧菜单要有ECU配置，点击后进入ECU列表页面

ECU列表页面，需要有新增、批量删除的功能

ECU也是网格化卡片展示的，显示名称name、款型type，卡片上需要有查看、编辑、删除的功能

点击新增、查看、编辑功能时都是个大弹窗，大弹窗里面有多个tab页，每个tab页都对应一种配置数据：ECU基础配置、转发表通信配置、CAN接口配置、LIN接口配置、ETH接口配置

ECU基础配置，转发表通信配置，这两个标签页，是固定配置输入窗口，其他接口配置标签页中需要有新增、删除接口的按钮，新增后生成一行配置需要用户输入

最终在这个大弹窗的右下角有保存、和取消按钮，保存时需要校验所有必填项目，并一次性创建所有配置并自动关联

如下配置，后端需要创建枚举，并提供查询接口，前端需要查询接口并展示为下拉框选取， 下拉框中展示的是枚举的名称，传给后端时是枚举的数字码
- CAN接口类型：0-CAN，1-CANFD
- CAN接口连接类型：0-MCU直连CAN，1-LSW下挂CAN
- ETH接口类型：0-百兆，1-千兆

### 数据定义

#### 1）ECU

| 字段        | 类型      | 含义      |
| --------- | ------- | ------- |
| id        | String  | 主键ID    |
| projectId | String  | 关联的工程ID |
| name      | String  | ECU设备名称 |
| desc      | String  | 描述      |
| mac       | String  | MAC地址   |
| ip        | String  | IP地址    |
| port      | Integer | 端口号     |
| index     | Integer | 部件索引号   |

对应前端表单 ： ECU基础配置

| 名称    | 对应字段  | 是否必填 |
| ----- | ----- | ---- |
| 名称    | name  | 是    |
| 描述    | desc  | 否    |
| MAC地址 | mac   | 是    |
| IP地址  | ip    | 是    |
| 端口号   | port  | 是    |
| 设备索引  | index | 是    |

校验规则：
- projectId必须在当前项目中存在
- 同project工程下的ECU设备名称不可重复
- mac地址要符合协议规范，同项目中的ECU的mac地址不可以重复
- ip地址要符合协议规范，同项目中的ECU的ip地址不可以重复
- port需要复合协议规范，最起码应是是大于等于0的整数，同项目中的ECU的port不可以重复
- index必须是大于等于0的整数，且同项目中的ECU的index不可以重复

#### 2） EcuForwardInfo

| 字段                       | 类型     | 含义             |
| ------------------------ | ------ | -------------- |
| id                       | String | 主键ID           |
| projectId                | String | 关联的工程ID        |
| ecuId                    | String | 关联的ECU 设备ID    |
| pFlashMemoryStartAddress | String | pFlash空间起始地址   |
| pFlashMemorySizeLimit    | String | pFlash空间编排大小限制 |
| ramMemoryStartAddress    | String | RAM空间起始地址      |
| ramMemorySizeLimit       | String | RAM空间S19编排大小限制 |

对应前端表单 ： 转发表通信配置

| 名称           | 对应字段                     | 是否必填 |
| ------------ | ------------------------ | ---- |
| pFlash空间起始地址 | pFlashMemoryStartAddress | 必填   |
| pFlash空间大小   | pFlashMemorySizeLimit    | 必填   |
| RAM空间起始地址    | ramMemoryStartAddress    | 必填   |
| RAM空间大小      | ramMemorySizeLimit       | 必填   |

校验规则：
- 4个字段都是大于0的十六进制数，并且要使用公共方法统一十六进制格式，即0x开头

#### 3）Interface

| 字段            | 类型      | 含义          |
| ------------- | ------- | ----------- |
| id            | String  | 主键ID        |
| projectId     | String  | 关联的工程ID     |
| ecuId         | String  | 关联的ECU 设备ID |
| interfaceName | String  | CAN接口名称     |
| channelId     | Integer | 接口通道ID      |
| port          | Integer | 端口号         |
#### 4）CanInterface extends Interface

| 字段       | 类型      | 含义                           |
| -------- | ------- | ---------------------------- |
| type     | Integer | 接口类型：0-CAN，1-CANFD           |
| connType | Integer | 接口连接类型：0-MCU直连CAN，1-LSW下挂CAN |

对应前端表单：CAN接口配置

| 名称        | 对应字段          | 是否必填 |
| --------- | ------------- | ---- |
| CAN接口名称   | interfaceName | 是    |
| CAN接口通道ID | channelId     | 是    |
| 端口号       | port          | 是    |
| CAN接口类型   | type          | 是    |
| CAN接口连接类型 | connType      | 是    |

校验规则：
- 同工程同ECU下的CAN、LIN、ETH接口名称不可重复
- 同工程同ECU下的CanInterface中的channelId不可重复，大于等于0的整数
- 同工程同ECU下的接口port不可重复，大于等于0的整数
-  CAN接口类型必须在枚举定义范围内
- CAN接口连接类型必须在枚举定义范围内


#### 5）LinInterface extends Interface

子类无新增字段

对应前端表单：Lin接口配置

| 名称      | 对应字段          | 是否必填校验 |
| ------- | ------------- | ------ |
| LIN接口名称 | interfaceName | 是      |
| 接口通道ID  | channelId     | 是      |
| 端口号     | port          | 是      |
校验规则：
- 同工程同ECU下的CAN、LIN、ETH接口名称不可重复
- 同工程同ECU下的LinInterface中的channelId不可重复，大于等于0的整数
- 同工程同ECU下的接口port不可重复，大于等于0的整数

#### 6）EthInterface extends Interface

| 字段   | 类型      | 含义                |
| ---- | ------- | ----------------- |
| type | Integer | ETH接口类型：0-百兆，1-千兆 |

对应前端表单：Eth接口配置

| 名称      | 对应字段          | 是否必填 |
| ------- | ------------- | ---- |
| ETH接口名称 | interfaceName | 是    |
| 接口通道ID  | channelId     | 是    |
| 端口号     | port          | 是    |
| ETH接口类型 | type          | 是    |
校验规则：
- 同工程同ECU下的CAN、LIN、ETH接口名称不可重复
- 同工程同ECU下的EthInterface中的channelId不可重复，大于等于0的整数
- 同工程同ECU下的接口port不可重复，大于等于0的整数
- ETH接口类型需要在枚举范围中
## 3. CAN/LIN通信配置

进入工程后，左侧菜单需要有 `CAN/LIN通信` 配置按钮

总线配置页面也需要有新增、批量删除的功能

总线以列表形式展现，每列都有编辑、删除的按钮功能

新增、编辑时，可在 新增列/当前列 展示输入框，然后输入必填配置后点击保存按钮进行数据入库

配置项和实例对象的定义：成
- id， 主键id，不可配置，无需前端展示，雪花算法自动生成
- subnet，String ，总线名称，必填
- interface，接口信息，下拉框选择，显示（`<ECU名称>:<CAN/LIN接口名称>`），下拉框的可选内容是通过后端接口查询的，将LinInterface和CanInterface列表查出来，然后用户进行选择之后，自动填充如下内容
	- busId，String，不可配置，