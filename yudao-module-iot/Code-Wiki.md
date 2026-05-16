# yudao-module-iot Code Wiki

## 一、项目概述

`yudao-module-iot` 是芋道（ruoyi-vue-pro）快速开发平台中的 **物联网（IoT）模块**，提供完整的设备管理、协议接入、消息通信、规则引擎、OTA 升级等核心 IoT 能力。模块采用分层架构设计，将业务逻辑（biz）、设备网关（gateway）和核心共享代码（core）解耦为三个独立子模块，支持独立部署和水平扩展。

### 核心能力

| 能力 | 说明 |
|------|------|
| 多协议接入 | 支持 MQTT、TCP、UDP、HTTP、WebSocket、CoAP、EMQX、Modbus TCP Client/Server 共 9 种协议 |
| 设备管理 | 设备注册/注销、状态管理、网关-子设备拓扑、设备分组、设备导入导出 |
| 产品管理 | 产品定义、产品分类、协议类型与序列化方式配置、一型一密动态注册 |
| 消息通信 | 基于消息总线的设备上行/下行消息流转，支持属性上报、事件上报、服务调用等 |
| 规则引擎 | 场景联动规则（触发器 + 条件 + 动作）、数据流转规则（数据源 → 数据目的） |
| OTA 升级 | 固件管理、升级任务、升级记录跟踪 |
| 告警管理 | 告警配置、告警记录、告警恢复 |
| 物模型 | 属性、事件、服务的物模型定义，TSL（Thing Specification Language）导出 |
| 数据存储 | MySQL（关系数据）+ TDengine（时序数据）+ Redis（缓存与消息总线） |

---

## 二、整体架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           外部设备 (Devices)                            │
│   MQTT设备 │ TCP设备 │ UDP设备 │ HTTP设备 │ WebSocket设备 │ CoAP设备 │ Modbus设备 │
└─────┬──────────┬──────────┬──────────┬──────────────┬──────────┬──────────┘
      │          │          │          │              │          │
      ▼          ▼          ▼          ▼              ▼          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     yudao-module-iot-gateway (设备网关)                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    IotProtocolManager (协议管理器)                │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────┐│  │
│  │  │ HTTP │ │ TCP  │ │ UDP  │ │ MQTT │ │ EMQX │ │ CoAP │ │WS  ││  │
│  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └─┬──┘│  │
│  └─────┼────────┼────────┼────────┼────────┼────────┼────────┼───┘  │
│        │        │        │        │        │        │        │       │
│  ┌─────▼────────▼────────▼────────▼────────▼────────▼────────▼───┐  │
│  │           IotMessageSerializer (序列化器 JSON/Binary)          │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │              IotMessageBus (消息总线)                          │  │
│  │        Local / Redis Stream / RocketMQ                        │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      yudao-module-iot-biz (业务主程序)                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                     Controller 层 (REST API)                      │ │
│  │  设备 │ 产品 │ OTA │ 告警 │ 统计 │ 规则 │ 物模型 │ 设备属性/消息  │ │
│  └───────────────────────────┬───────────────────────────────────────┘ │
│                              │                                         │
│  ┌───────────────────────────▼───────────────────────────────────────┐ │
│  │                       Service 层 (业务逻辑)                       │ │
│  │  IotDeviceService │ IotProductService │ IotSceneRuleService │ ... │ │
│  └───────────────────────────┬───────────────────────────────────────┘ │
│                              │                                         │
│  ┌───────────────────────────▼───────────────────────────────────────┐ │
│  │                        DAL 层 (数据访问)                          │ │
│  │  MySQL (MyBatis Plus) │ TDengine (时序) │ Redis (缓存)            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │              IoTDeviceApiImpl (RPC 接口, 供 Gateway 调用)         │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心数据流

#### 上行消息（设备 → 平台）

```
设备 → 协议(UpstreamHandler解码) → IotDeviceMessage → IotMessageBus.post()
    → IotDeviceMessageSubscriber.onMessage()
        → IotDeviceMessageService.handleUpstreamDeviceMessage()
            → 1. 处理消息(属性保存/状态更新/拓扑管理等)
            → 2. 记录消息(TDengine)
            → 3. 回复消息(如有必要)
            → 4. 触发规则引擎(场景联动/数据流转)
```

#### 下行消息（平台 → 设备）

```
业务操作 → IotDeviceMessageService.sendDeviceMessage()
    → IotDeviceMessageProducer.sendDeviceMessageToGateway(serverId)
        → IotMessageBus.post("iot_device_message_{serverId}")
            → AbstractIotProtocolDownstreamSubscriber.onMessage()
                → Protocol(DownstreamHandler编码) → 设备
```

---

## 三、子模块详解

### 3.1 yudao-module-iot-core（核心共享模块）

提供 biz 和 gateway 模块共用的核心功能，是整个 IoT 模块的基础层。

#### 包结构

```
cn.iocoder.yudao.module.iot.core
├── biz/                          # 业务通用 API 与 DTO
│   ├── IotDeviceCommonApi.java   # 设备通用 API 接口（认证/查询/注册/Modbus配置）
│   └── dto/                      # 通用 DTO
│       ├── IotDeviceAuthReqDTO.java
│       ├── IotDeviceRespDTO.java
│       ├── IotDeviceGetReqDTO.java
│       ├── IotModbusDeviceConfigRespDTO.java
│       ├── IotModbusPointRespDTO.java
│       ├── IotModbusDeviceConfigListReqDTO.java
│       └── IotSubDeviceRegisterFullReqDTO.java
├── enums/                        # 枚举定义
│   ├── IotProtocolTypeEnum.java       # 协议类型（TCP/UDP/MQTT/HTTP/CoAP/EMQX/WebSocket/Modbus）
│   ├── IotSerializeTypeEnum.java      # 序列化类型（JSON/Binary）
│   ├── IotDeviceMessageMethodEnum.java # 设备消息方法枚举（属性上报/事件/服务调用/拓扑/OTA等）
│   ├── device/
│   │   └── IotDeviceStateEnum.java    # 设备状态（在线/离线/未激活）
│   └── modbus/
│       ├── IotModbusModeEnum.java     # Modbus 模式
│       ├── IotModbusByteOrderEnum.java # 字节序
│       ├── IotModbusFrameFormatEnum.java # 帧格式
│       └── IotModbusRawDataTypeEnum.java # 原始数据类型
├── messagebus/                   # 消息总线
│   ├── config/
│   │   ├── IotMessageBusProperties.java       # 消息总线配置（type: local/redis/rocketmq）
│   │   └── IotMessageBusAutoConfiguration.java # 自动配置（按 type 条件装配）
│   └── core/
│       ├── IotMessageBus.java                 # 消息总线接口（post/register/unregister）
│       ├── IotMessageSubscriber.java          # 消息订阅者接口（getTopic/getGroup/onMessage）
│       ├── local/IotLocalMessageBus.java      # Local 实现（Spring Event）
│       ├── redis/IotRedisMessageBus.java      # Redis Stream 实现
│       └── rocketmq/IotRocketMQMessageBus.java # RocketMQ 实现
├── mq/                           # 消息队列封装
│   ├── message/
│   │   └── IotDeviceMessage.java  # 设备消息（核心数据结构）
│   └── producer/
│       └── IotDeviceMessageProducer.java # 设备消息生产者
├── topic/                        # Topic 相关 DTO
│   ├── IotDeviceIdentity.java     # 设备标识（productKey + deviceName）
│   ├── auth/                      # 设备注册相关 DTO
│   ├── ota/                       # OTA 升级相关 DTO
│   ├── state/                     # 设备状态相关 DTO
│   ├── topo/                      # 拓扑管理相关 DTO
│   ├── config/                    # 配置推送相关 DTO
│   ├── event/                     # 事件上报相关 DTO
│   ├── service/                   # 服务调用相关 DTO
│   └── property/                  # 属性上报/设置相关 DTO
└── util/                         # 工具类
    ├── IotDeviceAuthUtils.java    # 设备认证工具（用户名/密码生成与解析）
    ├── IotProductAuthUtils.java   # 产品认证工具（签名验证）
    └── IotDeviceMessageUtils.java # 设备消息工具（Topic构建/消息ID生成/上行判断）
```

#### 关键类说明

| 类名 | 职责 |
|------|------|
| `IotMessageBus` | 消息总线接口，定义 `post()` 发布消息和 `register()` 注册订阅者，是 biz 与 gateway 通信的核心桥梁 |
| `IotMessageSubscriber<T>` | 消息订阅者接口，定义 `getTopic()`、`getGroup()`、`onMessage()`，所有消息消费者需实现此接口 |
| `IotDeviceMessage` | 设备消息核心数据结构，包含 messageId、deviceId、tenantId、serverId、method、params、data 等字段 |
| `IotDeviceMessageProducer` | 消息生产者，封装 `IotMessageBus`，提供 `sendDeviceMessage()` 和 `sendDeviceMessageToGateway()` |
| `IotDeviceCommonApi` | 设备通用 API 接口，定义 `authDevice()`、`getDevice()`、`registerDevice()`、`registerSubDevices()`、`getModbusDeviceConfigList()` |
| `IotProtocolTypeEnum` | 协议类型枚举：TCP、UDP、WebSocket、HTTP、MQTT、EMQX、CoAP、MODBUS_TCP_CLIENT、MODBUS_TCP_SERVER |
| `IotDeviceMessageMethodEnum` | 消息方法枚举，定义了完整的 Alink 协议方法：属性上报/设置、事件上报、服务调用、拓扑管理、OTA、状态更新、设备注册等 |
| `IotDeviceIdentity` | 设备标识，由 `productKey` + `deviceName` 组成，是设备在系统中的唯一身份标识 |
| `IotMessageBusAutoConfiguration` | 消息总线自动配置，根据 `yudao.iot.message-bus.type` 配置选择 Local/Redis/RocketMQ 实现 |

#### 消息总线实现选择

| 类型 | 配置值 | 适用场景 | 说明 |
|------|--------|---------|------|
| Local | `local` | 单机开发/测试 | 基于 Spring Event，进程内通信，无额外依赖 |
| Redis | `redis` | 生产环境（推荐） | 基于 Redis Stream，支持持久化和消费者组 |
| RocketMQ | `rocketmq` | 高吞吐场景 | 基于 RocketMQ Topic，支持顺序消息和事务消息 |

---

### 3.2 yudao-module-iot-biz（业务主程序模块）

实现所有 IoT 业务逻辑，作为 `yudao-server` 主应用的一部分运行，提供 REST API 和 RPC 接口。

#### 包结构

```
cn.iocoder.yudao.module.iot
├── api/                           # RPC API 实现
│   └── device/
│       └── IoTDeviceApiImpl.java  # 设备 API 实现（供 Gateway 通过 HTTP RPC 调用）
├── controller/                    # Controller 层
│   └── admin/
│       ├── device/                # 设备管理
│       │   ├── IotDeviceController.java
│       │   ├── IotDeviceGroupController.java
│       │   ├── IotDeviceMessageController.java
│       │   ├── IotDevicePropertyController.java
│       │   ├── IotDeviceModbusConfigController.java
│       │   ├── IotDeviceModbusPointController.java
│       │   └── vo/               # 请求/响应 VO
│       ├── product/               # 产品管理
│       │   ├── IotProductController.java
│       │   ├── IotProductCategoryController.java
│       │   └── vo/
│       ├── ota/                   # OTA 升级
│       │   ├── IotOtaFirmwareController.java
│       │   ├── IotOtaTaskController.java
│       │   ├── IotOtaTaskRecordController.java
│       │   └── vo/
│       ├── alert/                 # 告警管理
│       │   ├── IotAlertConfigController.java
│       │   ├── IotAlertRecordController.java
│       │   └── vo/
│       ├── rule/                  # 规则引擎
│       │   ├── IotSceneRuleController.java
│       │   ├── IotDataRuleController.java
│       │   ├── IotDataSinkController.java
│       │   └── vo/
│       ├── statistics/            # 统计分析
│       │   ├── IotStatisticsController.java
│       │   └── vo/
│       └── thingmodel/            # 物模型
│           ├── IotThingModelController.java
│           └── vo/
├── convert/                       # 对象转换
│   └── thingmodel/
│       └── IotThingModelConvert.java
├── dal/                           # 数据访问层
│   ├── dataobject/               # 数据对象
│   │   ├── device/
│   │   │   ├── IotDeviceDO.java
│   │   │   ├── IotDeviceGroupDO.java
│   │   │   ├── IotDeviceMessageDO.java
│   │   │   ├── IotDevicePropertyDO.java
│   │   │   ├── IotDeviceModbusConfigDO.java
│   │   │   └── IotDeviceModbusPointDO.java
│   │   ├── product/
│   │   │   ├── IotProductDO.java
│   │   │   └── IotProductCategoryDO.java
│   │   ├── ota/
│   │   │   ├── IotOtaFirmwareDO.java
│   │   │   ├── IotOtaTaskDO.java
│   │   │   └── IotOtaTaskRecordDO.java
│   │   ├── alert/
│   │   │   ├── IotAlertConfigDO.java
│   │   │   └── IotAlertRecordDO.java
│   │   ├── rule/
│   │   │   ├── IotSceneRuleDO.java
│   │   │   ├── IotDataRuleDO.java
│   │   │   ├── IotDataSinkDO.java
│   │   │   └── config/           # 数据目的配置
│   │   │       ├── IotAbstractDataSinkConfig.java
│   │   │       ├── IotDataSinkHttpConfig.java
│   │   │       ├── IotDataSinkMqttConfig.java
│   │   │       ├── IotDataSinkKafkaConfig.java
│   │   │       ├── IotDataSinkRabbitMQConfig.java
│   │   │       ├── IotDataSinkRocketMQConfig.java
│   │   │       ├── IotDataSinkRedisConfig.java
│   │   │       ├── IotDataSinkDatabaseConfig.java
│   │   │       ├── IotDataSinkWebSocketConfig.java
│   │   │       └── IotDataSinkTcpConfig.java
│   │   └── thingmodel/
│   │       ├── IotThingModelDO.java
│   │       └── model/            # 物模型定义
│   │           ├── ThingModelProperty.java
│   │           ├── ThingModelEvent.java
│   │           ├── ThingModelService.java
│   │           ├── ThingModelParam.java
│   │           └── dataType/     # 数据类型
│   ├── mysql/                    # MySQL Mapper
│   │   ├── device/
│   │   ├── product/
│   │   ├── ota/
│   │   ├── alert/
│   │   ├── rule/
│   │   └── thingmodel/
│   ├── tdengine/                 # TDengine Mapper（时序数据）
│   │   ├── IotDeviceMessageMapper.java
│   │   └── IotDevicePropertyMapper.java
│   └── redis/                    # Redis DAO
│       ├── RedisKeyConstants.java
│       └── device/
│           ├── DeviceServerIdRedisDAO.java
│           ├── DevicePropertyRedisDAO.java
│           └── DeviceReportTimeRedisDAO.java
├── framework/                     # 框架扩展
│   ├── iot/
│   │   └── config/
│   │       └── YudaoIotProperties.java  # IoT 全局配置（keepAliveTime 等）
│   ├── tdengine/
│   │   ├── config/
│   │   │   └── TDengineTableInitRunner.java  # TDengine 表初始化
│   │   ├── core/
│   │   │   ├── TDengineTableField.java
│   │   │   └── annotation/
│   │   │       └── TDengineDS.java
│   │   └── package-info.java
│   ├── web/
│   │   └── config/
│   │       └── IotWebConfiguration.java
│   └── job/
│       ├── config/
│       │   └── IotJobConfiguration.java
│       └── core/
│           └── IotSchedulerManager.java
├── mq/                            # 消息消费者
│   └── consumer/
│       ├── device/
│       │   └── IotDeviceMessageSubscriber.java  # 设备消息消费者
│       └── rule/
│           ├── IotSceneRuleMessageSubscriber.java  # 场景规则消息消费者
│           └── IotDataRuleMessageSubscriber.java   # 数据规则消息消费者
└── service/                       # Service 层
    ├── device/
    │   ├── IotDeviceService.java / IotDeviceServiceImpl.java
    │   ├── IotDeviceGroupService.java / IotDeviceGroupServiceImpl.java
    │   ├── IotDeviceMessageService.java / IotDeviceMessageServiceImpl.java
    │   ├── IotDevicePropertyService.java / IotDevicePropertyServiceImpl.java
    │   ├── IotDeviceModbusConfigService.java / IotDeviceModbusConfigServiceImpl.java
    │   └── IotDeviceModbusPointService.java / IotDeviceModbusPointServiceImpl.java
    ├── product/
    │   ├── IotProductService.java / IotProductServiceImpl.java
    │   └── IotProductCategoryService.java / IotProductCategoryServiceImpl.java
    ├── ota/
    │   ├── IotOtaFirmwareService.java / IotOtaFirmwareServiceImpl.java
    │   ├── IotOtaTaskService.java / IotOtaTaskServiceImpl.java
    │   └── IotOtaTaskRecordService.java / IotOtaTaskRecordServiceImpl.java
    ├── alert/
    │   ├── IotAlertConfigService.java / IotAlertConfigServiceImpl.java
    │   └── IotAlertRecordService.java / IotAlertRecordServiceImpl.java
    ├── rule/
    │   ├── scene/
    │   │   ├── IotSceneRuleService.java / IotSceneRuleServiceImpl.java
    │   │   ├── IotSceneRuleTimeHelper.java
    │   │   ├── matcher/           # 场景匹配器
    │   │   │   ├── IotSceneRuleMatcherManager.java
    │   │   │   ├── IotSceneRuleMatcher.java
    │   │   │   ├── IotSceneRuleMatcherHelper.java
    │   │   │   ├── trigger/      # 触发器匹配器
    │   │   │   │   ├── IotSceneRuleTriggerMatcher.java
    │   │   │   │   ├── IotDevicePropertyPostTriggerMatcher.java
    │   │   │   │   ├── IotDeviceEventPostTriggerMatcher.java
    │   │   │   │   ├── IotDeviceStateUpdateTriggerMatcher.java
    │   │   │   │   ├── IotDeviceServiceInvokeTriggerMatcher.java
    │   │   │   │   └── IotTimerTriggerMatcher.java
    │   │   │   └── condition/    # 条件匹配器
    │   │   │       ├── IotSceneRuleConditionMatcher.java
    │   │   │       ├── IotDevicePropertyConditionMatcher.java
    │   │   │       ├── IotDeviceStateConditionMatcher.java
    │   │   │       └── IotCurrentTimeConditionMatcher.java
    │   │   ├── action/           # 场景动作执行器
    │   │   │   ├── IotSceneRuleAction.java
    │   │   │   ├── IotDevicePropertySetSceneRuleAction.java
    │   │   │   ├── IotDeviceServiceInvokeSceneRuleAction.java
    │   │   │   ├── IotAlertTriggerSceneRuleAction.java
    │   │   │   └── IotAlertRecoverSceneRuleAction.java
    │   │   └── timer/           # 定时触发器
    │   │       ├── IotSceneRuleTimerHandler.java
    │   │       └── IotTimerConditionEvaluator.java
    │   └── data/
    │       ├── IotDataRuleService.java / IotDataRuleServiceImpl.java
    │       ├── IotDataSinkService.java / IotDataSinkServiceImpl.java
    │       └── action/           # 数据流转动作执行器
    │           ├── IotDataRuleAction.java
    │           ├── IotDataRuleCacheableAction.java
    │           ├── IotHttpDataSinkAction.java
    │           ├── IotMqttDataRuleAction.java
    │           ├── IotKafkaDataRuleAction.java
    │           ├── IotRabbitMQDataRuleAction.java
    │           ├── IotRocketMQDataRuleAction.java
    │           ├── IotRedisRuleAction.java
    │           ├── IotDatabaseDataRuleAction.java
    │           ├── IotWebSocketDataRuleAction.java
    │           ├── IotTcpDataRuleAction.java
    │           ├── websocket/IotWebSocketClient.java
    │           └── tcp/IotTcpClient.java
    └── thingmodel/
        └── IotThingModelService.java / IotThingModelServiceImpl.java
```

#### 关键 Service 说明

| Service | 职责 | 核心方法 |
|---------|------|---------|
| `IotDeviceService` | 设备全生命周期管理 | `createDevice()`, `updateDeviceState()`, `authDevice()`, `bindDeviceGateway()`, `handleTopoAddMessage()`, `registerDevice()` |
| `IotProductService` | 产品管理 | `createProduct()`, `getProductByProductKey()`, `getProductFromCache()` |
| `IotDeviceMessageService` | 设备消息处理 | `sendDeviceMessage()`, `handleUpstreamDeviceMessage()`, `defineDeviceMessageStable()` |
| `IotDevicePropertyService` | 设备属性数据 | `saveDeviceProperty()`, `getLatestDeviceProperties()`, `getHistoryDevicePropertyList()`, `updateDeviceReportTimeAsync()` |
| `IotSceneRuleService` | 场景联动规则 | `createSceneRule()`, `executeSceneRuleByDevice()`, `executeSceneRuleByTimer()` |
| `IotDataRuleService` | 数据流转规则 | `createDataRule()`, `executeDataRule()` |
| `IotDataSinkService` | 数据目的管理 | `createDataSink()`, `getDataSinkList()` |
| `IotOtaFirmwareService` | OTA 固件管理 | `createOtaFirmware()`, `getOtaFirmwarePage()` |
| `IotOtaTaskService` | OTA 升级任务 | `createOtaTask()`, `getOtaTaskPage()` |
| `IotAlertConfigService` | 告警配置 | `createAlertConfig()`, `getAlertConfigPage()` |
| `IotAlertRecordService` | 告警记录 | `createAlertRecord()`, `processAlertRecord()` |
| `IotThingModelService` | 物模型管理 | `createThingModel()`, `getThingModelList()`, `getThingModelTSL()` |

#### 关键数据对象

| DO | 对应表 | 说明 |
|----|--------|------|
| `IotDeviceDO` | `iot_device` | 设备，包含 deviceName、productKey、deviceSecret、state、gatewayId、groupIds、firmwareId、longitude/latitude 等 |
| `IotProductDO` | `iot_product` | 产品，包含 productKey、productSecret、registerEnabled、deviceType、protocolType、serializeType 等 |
| `IotProductCategoryDO` | `iot_product_category` | 产品分类 |
| `IotDeviceGroupDO` | `iot_device_group` | 设备分组 |
| `IotDeviceMessageDO` | TDengine 超级表 | 设备消息（时序数据） |
| `IotDevicePropertyDO` | TDengine 超级表 | 设备属性（时序数据） |
| `IotDeviceModbusConfigDO` | `iot_device_modbus_config` | Modbus 设备配置 |
| `IotDeviceModbusPointDO` | `iot_device_modbus_point` | Modbus 点位配置 |
| `IotSceneRuleDO` | `iot_scene_rule` | 场景联动规则，内嵌 Trigger/TriggerCondition/Action |
| `IotDataRuleDO` | `iot_data_rule` | 数据流转规则，内嵌 SourceConfig |
| `IotDataSinkDO` | `iot_data_sink` | 数据目的，内嵌 IotAbstractDataSinkConfig |
| `IotOtaFirmwareDO` | `iot_ota_firmware` | OTA 固件 |
| `IotOtaTaskDO` | `iot_ota_task` | OTA 升级任务 |
| `IotOtaTaskRecordDO` | `iot_ota_task_record` | OTA 升级记录 |
| `IotAlertConfigDO` | `iot_alert_config` | 告警配置 |
| `IotAlertRecordDO` | `iot_alert_record` | 告警记录 |
| `IotThingModelDO` | `iot_thing_model` | 物模型定义 |

#### 规则引擎架构

**场景联动规则**（IotSceneRuleDO）采用 **触发器 → 条件 → 动作** 模型：

```
Trigger（触发器）
├── DEVICE_STATE_UPDATE   # 设备状态变更
├── DEVICE_PROPERTY_POST  # 属性上报
├── DEVICE_EVENT_POST     # 事件上报
├── DEVICE_SERVICE_INVOKE # 服务调用
└── TIMER                 # 定时触发（CRON 表达式）
    │
    ├── conditionGroups（条件分组，组间"或"，组内"且"）
    │   ├── DEVICE_STATE    # 设备状态条件
    │   ├── DEVICE_PROPERTY # 属性条件
    │   └── CURRENT_TIME    # 时间条件
    │
    └── Action（执行动作）
        ├── DEVICE_PROPERTY_SET  # 设置设备属性
        ├── DEVICE_SERVICE_INVOKE # 调用设备服务
        ├── ALERT_TRIGGER        # 触发告警
        └── ALERT_RECOVER        # 恢复告警
```

**数据流转规则**（IotDataRuleDO）采用 **数据源 → 数据目的** 模型：

```
SourceConfig（数据源）
├── method     # 监听的消息方法（如 thing.property.post）
├── productId  # 产品过滤
├── deviceId   # 设备过滤（0=全部设备）
└── identifier # 标识符过滤
    │
    └── DataSink（数据目的）
        ├── HTTP      # HTTP 推送
        ├── MQTT      # MQTT 发布
        ├── Kafka     # Kafka 发送
        ├── RabbitMQ  # RabbitMQ 发送
        ├── RocketMQ  # RocketMQ 发送
        ├── Redis     # Redis 存储
        ├── Database  # 数据库存储
        ├── WebSocket # WebSocket 推送
        └── TCP       # TCP 推送
```

---

### 3.3 yudao-module-iot-gateway（设备网关模块）

独立 Spring Boot 应用，负责设备协议接入和消息编解码，是设备与平台之间的桥梁。

#### 包结构

```
cn.iocoder.yudao.module.iot.gateway
├── IotGatewayServerApplication.java   # 网关启动类
├── config/
│   ├── IotGatewayConfiguration.java   # 网关配置类（注册 ProtocolManager、SerializerManager）
│   └── IotGatewayProperties.java      # 网关配置属性（RPC/Token/协议实例列表）
├── serialize/                          # 序列化器
│   ├── IotMessageSerializer.java       # 序列化器接口
│   ├── IotMessageSerializerManager.java # 序列化器管理器
│   ├── json/IotJsonSerializer.java     # JSON 序列化实现
│   └── binary/IotBinarySerializer.java # Binary 序列化实现
├── protocol/                           # 协议实现
│   ├── IotProtocol.java               # 协议接口（getId/getServerId/getType/start/stop/isRunning）
│   ├── IotProtocolManager.java        # 协议管理器（根据配置创建/启停协议实例）
│   ├── AbstractIotProtocolDownstreamSubscriber.java # 下行消息订阅抽象类
│   ├── http/                          # HTTP 协议
│   │   ├── IotHttpProtocol.java
│   │   ├── IotHttpConfig.java
│   │   └── handler/
│   │       ├── upstream/              # 上行处理（认证/注册/消息解析）
│   │       └── downstream/            # 下行处理（消息编码发送）
│   ├── tcp/                           # TCP 协议
│   │   ├── IotTcpProtocol.java
│   │   ├── IotTcpConfig.java
│   │   ├── codec/                     # TCP 编解码（分隔符/长度字段/固定长度）
│   │   ├── handler/
│   │   │   ├── upstream/
│   │   │   └── downstream/
│   │   └── manager/
│   │       └── IotTcpConnectionManager.java
│   ├── udp/                           # UDP 协议
│   │   ├── IotUdpProtocol.java
│   │   ├── IotUdpConfig.java
│   │   ├── handler/
│   │   └── manager/
│   │       └── IotUdpSessionManager.java
│   ├── websocket/                     # WebSocket 协议
│   │   ├── IotWebSocketProtocol.java
│   │   ├── IotWebSocketConfig.java
│   │   ├── handler/
│   │   └── manager/
│   │       └── IotWebSocketConnectionManager.java
│   ├── mqtt/                          # MQTT 协议（自建 Broker）
│   │   ├── IotMqttProtocol.java
│   │   ├── IotMqttConfig.java
│   │   ├── handler/
│   │   │   ├── upstream/
│   │   │   │   ├── IotMqttAuthHandler.java       # MQTT 认证处理
│   │   │   │   ├── IotMqttRegisterHandler.java   # MQTT 注册处理
│   │   │   │   └── IotMqttUpstreamHandler.java   # MQTT 消息上行处理
│   │   │   └── downstream/
│   │   └── manager/
│   │       └── IotMqttConnectionManager.java
│   ├── emqx/                          # EMQX 协议（对接 EMQX Broker）
│   │   ├── IotEmqxProtocol.java
│   │   ├── IotEmqxConfig.java
│   │   └── handler/
│   │       ├── upstream/
│   │       │   ├── IotEmqxAuthEventHandler.java  # EMQX HTTP Hook 认证
│   │       │   └── IotEmqxUpstreamHandler.java   # EMQX 消息上行
│   │       └── downstream/
│   ├── coap/                          # CoAP 协议
│   │   ├── IotCoapProtocol.java
│   │   ├── IotCoapConfig.java
│   │   └── handler/
│   │       ├── upstream/
│   │       │   ├── IotCoapAuthHandler.java
│   │       │   ├── IotCoapRegisterHandler.java
│   │       │   └── IotCoapUpstreamHandler.java
│   │       └── downstream/
│   └── modbus/                        # Modbus 协议
│       ├── tcpclient/                 # Modbus TCP Client（主动轮询）
│       │   ├── IotModbusTcpClientProtocol.java
│       │   ├── IotModbusTcpClientConfig.java
│       │   ├── handler/
│       │   └── manager/
│       │       ├── IotModbusTcpClientConnectionManager.java
│       │       ├── IotModbusTcpClientConfigCacheService.java
│       │       └── IotModbusTcpClientPollScheduler.java
│       ├── tcpserver/                 # Modbus TCP Server（被动响应）
│       │   ├── IotModbusTcpServerProtocol.java
│       │   ├── IotModbusTcpServerConfig.java
│       │   ├── codec/                 # Modbus 帧编解码
│       │   ├── handler/
│       │   └── manager/
│       │       ├── IotModbusTcpServerConnectionManager.java
│       │       ├── IotModbusTcpServerConfigCacheService.java
│       │       ├── IotModbusTcpServerPollScheduler.java
│       │       └── IotModbusTcpServerPendingRequestManager.java
│       └── common/                    # Modbus 公共代码
│           ├── manager/
│           │   └── AbstractIotModbusPollScheduler.java
│           └── utils/
└── enums/
    └── ErrorCodeConstants.java        # 网关错误码
```

#### 协议实现统一模式

每个协议实现遵循相同的架构模式：

```
IotProtocol (接口)
├── start() / stop() / isRunning()     # 生命周期管理
├── UpstreamHandler                     # 上行消息处理（设备→平台）
│   ├── 认证处理（Auth）               # 设备连接认证
│   ├── 注册处理（Register）           # 设备动态注册
│   └── 消息处理（Upstream）           # 解码设备消息，发布到消息总线
├── DownstreamHandler                   # 下行消息处理（平台→设备）
│   └── 编码消息，发送给设备
├── DownstreamSubscriber                # 下行消息订阅
│   └── 继承 AbstractIotProtocolDownstreamSubscriber
│       └── 订阅 iot_device_message_{serverId} Topic
└── ConnectionManager / SessionManager  # 连接/会话管理
```

#### 协议实例配置

网关通过 `yudao.iot.gateway.protocols` 配置列表支持多协议实例，每个实例包含：

| 配置项 | 说明 |
|--------|------|
| `id` | 协议实例 ID（如 "http-json"、"tcp-binary"） |
| `enabled` | 是否启用 |
| `protocol` | 协议类型（对应 `IotProtocolTypeEnum`） |
| `port` | 服务端口 |
| `serialize` | 序列化类型（json/binary，可选） |
| `ssl` | SSL 配置（可选） |
| `http/tcp/udp/websocket/mqtt/emqx/coap/modbusTcpClient/modbusTcpServer` | 各协议特有配置 |

---

## 四、依赖关系

### 4.1 模块间依赖

```
yudao-module-iot-biz ──→ yudao-module-iot-core
                      ──→ yudao-module-system
                      ──→ yudao-spring-boot-starter-web
                      ──→ yudao-spring-boot-starter-security
                      ──→ yudao-spring-boot-starter-mybatis
                      ──→ yudao-spring-boot-starter-redis
                      ──→ yudao-spring-boot-starter-biz-tenant
                      ──→ yudao-spring-boot-starter-excel
                      ──→ yudao-spring-boot-starter-test
                      ──→ taos-jdbcdriver (TDengine)
                      ──→ okhttp
                      ──→ paho.mqttv3
                      ──→ rocketmq-spring-boot-starter
                      ──→ spring-kafka
                      ──→ spring-boot-starter-amqp

yudao-module-iot-gateway ──→ yudao-module-iot-core
                          ──→ spring-web
                          ──→ vertx-web
                          ──→ vertx-mqtt
                          ──→ j2mod (Modbus)
                          ──→ californium-core (CoAP)
                          ──→ rocketmq-spring-boot-starter (optional)
                          ──→ yudao-spring-boot-starter-test

yudao-module-iot-core ──→ yudao-common
                      ──→ spring-boot-starter
                      ──→ yudao-spring-boot-starter-mq
                      ──→ spring-data-redis (optional)
                      ──→ rocketmq-spring-boot-starter (optional)
                      ──→ spring-rabbit (optional)
                      ──→ spring-kafka (optional)
```

### 4.2 外部中间件依赖

| 中间件 | 用途 | 必需 |
|--------|------|------|
| MySQL | 关系数据存储（设备/产品/规则等） | 是 |
| Redis | 缓存 + 消息总线（可选） | 是 |
| TDengine | 设备消息/属性时序数据存储 | 是 |
| RocketMQ | 消息总线（可选，替代 Redis） | 否 |
| RabbitMQ | 数据流转目的 + 消息总线 | 否 |
| Kafka | 数据流转目的 + 消息总线 | 否 |
| EMQX | MQTT Broker（使用 EMQX 协议时需要） | 否 |

### 4.3 核心框架依赖

| 框架 | 用途 |
|------|------|
| Spring Boot | 基础框架 |
| Spring Security | 安全认证与权限控制 |
| MyBatis Plus | ORM 框架 |
| Vert.x | 网关高性能网络通信（TCP/UDP/MQTT/WebSocket/HTTP） |
| Eclipse Paho | MQTT v3 客户端 |
| j2mod | Modbus 协议实现 |
| Eclipse Californium | CoAP 协议实现 |
| OkHttp | HTTP 客户端（RPC 调用） |

---

## 五、项目运行方式

### 5.1 主程序（yudao-server）

`yudao-module-iot-biz` 作为 `yudao-server` 的依赖模块运行，不需要单独启动。

**启动步骤：**

1. 准备中间件：MySQL、Redis、TDengine
2. 初始化数据库：执行 `sql/mysql/ruoyi-vue-pro.sql` 中的 IoT 相关表
3. 配置 `yudao-server/src/main/resources/application-local.yaml`：
   - MySQL 数据源
   - Redis 连接
   - TDengine 连接
   - `yudao.iot.message-bus.type` 消息总线类型
4. 启动 `YudaoServerApplication`

**关键配置项：**

```yaml
yudao:
  iot:
    message-bus:
      type: redis  # 消息总线类型：local / redis / rocketmq
```

### 5.2 设备网关（yudao-module-iot-gateway）

`yudao-module-iot-gateway` 是独立的 Spring Boot 应用，需要单独启动。

**启动步骤：**

1. 确保 `yudao-server` 主程序已启动
2. 配置 `application.yaml` 或 `application-local.yaml`：
   - Redis 连接
   - `yudao.iot.gateway.rpc.url` 指向主程序 API 地址
   - `yudao.iot.gateway.token` 配置 Token 密钥
   - `yudao.iot.gateway.protocols` 配置需要启用的协议实例
3. 启动 `IotGatewayServerApplication`

**关键配置项：**

```yaml
yudao:
  iot:
    message-bus:
      type: redis  # 必须与主程序一致
    gateway:
      rpc:
        url: http://127.0.0.1:48080  # 主程序 API 地址
        connect-timeout: 30s
        read-timeout: 30s
      token:
        secret: yudaoIotGatewayTokenSecret123456789  # Token 密钥（至少32位）
        expiration: 7d
      protocols:
        - id: mqtt-json
          protocol: mqtt
          port: 1883
          enabled: true
          serialize: json
```

### 5.3 多网关部署

支持部署多个网关实例实现水平扩展：

- 每个网关实例通过 `serverId`（即协议实例的 `id`）标识
- 设备连接时，网关将 `serverId` 记录到 Redis（`DeviceServerIdRedisDAO`）
- 下行消息通过 `iot_device_message_{serverId}` Topic 路由到对应网关
- 不同网关实例可以启用不同的协议，实现协议层面的水平扩展

---

## 六、核心流程详解

### 6.1 设备认证流程

```
设备连接 → Gateway 协议层（AuthHandler）
    → 解析 username 获取 productKey + deviceName（IotDeviceAuthUtils.parseUsername）
    → RPC 调用 IoTDeviceApiImpl.authDevice()
        → IotDeviceService.authDevice()
            → 从缓存获取设备信息
            → 校验密码（IotDeviceAuthUtils.getAuthInfo 生成预期密码，比对）
    → 认证成功/失败
```

### 6.2 设备动态注册流程（一型一密）

```
设备发送注册请求 → Gateway
    → RPC 调用 IoTDeviceApiImpl.registerDevice()
        → IotDeviceService.registerDevice()
            → 校验产品是否存在且开启动态注册
            → 验证签名（IotProductAuthUtils.verifySign）
            → 自动创建设备（生成 deviceSecret）
            → 返回 deviceSecret
```

### 6.3 属性上报流程

```
设备上报属性 → Gateway UpstreamHandler
    → 反序列化（IotMessageSerializer.deserialize）
    → 构建 IotDeviceMessage（method=thing.property.post）
    → IotMessageBus.post("iot_device_message")
        → IotDeviceMessageSubscriber.onMessage()
            → 强制设备上线（如未在线）
            → IotDeviceMessageService.handleUpstreamDeviceMessage()
                → 更新设备上报时间
                → 保存属性数据（TDengine）
                → 记录消息日志（TDengine）
                → 回复消息（如需要）
                → 触发场景联动规则
                → 触发数据流转规则
```

### 6.4 网关-子设备拓扑管理

```
网关设备上报 topo.add → Gateway
    → IotDeviceMessageSubscriber → IotDeviceMessageService
        → IotDeviceService.handleTopoAddMessage()
            → 解析子设备认证信息
            → 逐个认证子设备
            → 绑定子设备到网关（更新 gatewayId）
            → 返回绑定结果
```

### 6.5 场景联动执行流程

```
设备消息到达 → IotSceneRuleMessageSubscriber.onMessage()
    → IotSceneRuleService.executeSceneRuleByDevice()
        → 获取设备关联的场景规则列表（缓存）
        → 遍历规则，执行匹配：
            → TriggerMatcher 匹配触发器
            → ConditionMatcher 匹配条件
            → 匹配成功 → 执行 Action：
                ├── IotDevicePropertySetSceneRuleAction  # 下发属性设置
                ├── IotDeviceServiceInvokeSceneRuleAction # 下发服务调用
                ├── IotAlertTriggerSceneRuleAction       # 触发告警
                └── IotAlertRecoverSceneRuleAction       # 恢复告警
```

### 6.6 数据流转执行流程

```
设备消息到达 → IotDataRuleMessageSubscriber.onMessage()
    → IotDataRuleService.executeDataRule()
        → 匹配数据规则（SourceConfig 过滤 method/productId/deviceId/identifier）
        → 遍历匹配规则的 sinkIds
        → 执行数据目的动作：
            ├── IotHttpDataSinkAction     # HTTP 推送
            ├── IotMqttDataRuleAction     # MQTT 发布
            ├── IotKafkaDataRuleAction    # Kafka 发送
            ├── IotRabbitMQDataRuleAction # RabbitMQ 发送
            ├── IotRocketMQDataRuleAction # RocketMQ 发送
            ├── IotRedisRuleAction        # Redis 存储
            ├── IotDatabaseDataRuleAction # 数据库存储
            ├── IotWebSocketDataRuleAction # WebSocket 推送
            └── IotTcpDataRuleAction      # TCP 推送
```

---

## 七、设备消息方法（Alink 协议）

系统采用类似阿里云 IoT 的 Alink 协议定义设备消息方法：

| 方法 | 枚举值 | 方向 | 说明 |
|------|--------|------|------|
| `thing.state.update` | STATE_UPDATE | 上行 | 设备状态更新（上线/离线） |
| `thing.topo.add` | TOPO_ADD | 上行 | 添加拓扑关系（网关绑定子设备） |
| `thing.topo.delete` | TOPO_DELETE | 上行 | 删除拓扑关系 |
| `thing.topo.get` | TOPO_GET | 上行 | 获取拓扑关系 |
| `thing.topo.change` | TOPO_CHANGE | 下行 | 拓扑关系变更通知 |
| `thing.auth.register` | DEVICE_REGISTER | 上行 | 设备动态注册（一型一密） |
| `thing.auth.register.sub` | SUB_DEVICE_REGISTER | 上行 | 子设备动态注册 |
| `thing.property.post` | PROPERTY_POST | 上行 | 属性上报 |
| `thing.property.set` | PROPERTY_SET | 下行 | 属性设置 |
| `thing.event.property.pack.post` | PROPERTY_PACK_POST | 上行 | 批量上报（网关独有） |
| `thing.event.post` | EVENT_POST | 上行 | 事件上报 |
| `thing.service.invoke` | SERVICE_INVOKE | 下行 | 服务调用 |
| `thing.config.push` | CONFIG_PUSH | 下行 | 配置推送 |
| `thing.ota.upgrade` | OTA_UPGRADE | 下行 | OTA 固件信息推送 |
| `thing.ota.progress` | OTA_PROGRESS | 上行 | OTA 升级进度上报 |

---

## 八、缓存策略

| 缓存 Key | 说明 | 过期策略 |
|----------|------|---------|
| `iot:device:{id}` | 按 ID 查设备 | 主动失效（更新/删除时清除） |
| `iot:device:{productKey}_{deviceName}` | 按 productKey+deviceName 查设备 | 主动失效 |
| `iot:thing_model_list:{productId}` | 产品的物模型列表 | 主动失效 |
| `iot:device:server_id:{deviceId}` | 设备关联的网关 serverId | 异步更新 |
| `iot:device:property:{deviceId}` | 设备最新属性数据 | 异步更新 |
| `iot:device:report_time:{deviceId}` | 设备最后上报时间 | 异步更新 |

---

## 九、API 接口概览

### 9.1 管理后台 API（Controller 层）

| 模块 | 路径前缀 | 主要接口 |
|------|---------|---------|
| 设备管理 | `/iot/device/` | CRUD、导入导出、绑定/解绑网关、获取认证信息、设备位置 |
| 设备分组 | `/iot/device-group/` | CRUD |
| 设备消息 | `/iot/device-message/` | 分页查询、发送消息 |
| 设备属性 | `/iot/device-property/` | 最新属性、历史属性 |
| Modbus 配置 | `/iot/device-modbus-config/` | CRUD |
| Modbus 点位 | `/iot/device-modbus-point/` | CRUD |
| 产品管理 | `/iot/product/` | CRUD、产品状态更新 |
| 产品分类 | `/iot/product-category/` | CRUD |
| OTA 固件 | `/iot/ota-firmware/` | CRUD |
| OTA 任务 | `/iot/ota-task/` | 创建、分页 |
| OTA 记录 | `/iot/ota-task-record/` | 分页查询 |
| 告警配置 | `/iot/alert-config/` | CRUD |
| 告警记录 | `/iot/alert-record/` | 分页、处理 |
| 场景规则 | `/iot/scene-rule/` | CRUD、状态更新 |
| 数据规则 | `/iot/data-rule/` | CRUD |
| 数据目的 | `/iot/data-sink/` | CRUD |
| 统计分析 | `/iot/statistics/` | 概览统计、消息统计 |
| 物模型 | `/iot/thing-model/` | CRUD、TSL 导出 |

### 9.2 RPC API（供 Gateway 调用）

| 路径 | 方法 | 说明 |
|------|------|------|
| `/rpc-api/iot/device/auth` | POST | 设备认证 |
| `/rpc-api/iot/device/get` | POST | 获取设备信息 |
| `/rpc-api/iot/device/register` | POST | 设备动态注册 |
| `/rpc-api/iot/device/register-sub` | POST | 子设备动态注册 |
| `/rpc-api/iot/modbus/config-list` | POST | 获取 Modbus 设备配置列表 |

---

## 十、技术栈总结

| 层次 | 技术选型 |
|------|---------|
| 基础框架 | Spring Boot 2.x |
| Web 框架 | Spring MVC |
| 安全框架 | Spring Security |
| ORM 框架 | MyBatis Plus |
| 时序数据库 | TDengine |
| 缓存 | Redis (Spring Data Redis + Redisson) |
| 消息队列 | RocketMQ / Kafka / RabbitMQ (可选) |
| 网络框架 | Vert.x (TCP/UDP/MQTT/WebSocket/HTTP) |
| Modbus | j2mod |
| CoAP | Eclipse Californium |
| MQTT 客户端 | Eclipse Paho |
| HTTP 客户端 | OkHttp |
| 接口文档 | Swagger 3 / Knife4j |
| 对象转换 | MapStruct / BeanUtils |
| JSON | Jackson |
| 工具类 | Hutool |
