# Gateway 网关开发文档

基本要求：

1. 网关与云端的通讯协议采用 MQTT 协议。
2. 网关支持 4G 通讯模块，支持全网通网络。
3. 网关至少支持 1 路 LoRa 收发模块。
4. 网关支持蓝牙模块，要求支持LE功率控制，建议采用 5.2 版本的蓝牙模块。
5. 网关支持电池供电和电源供电。

## 程序写入流程
 
1. 通过 ST-LINK 写入 BootLoader 和 主系统。
2. 通过 ST-LINK 写入 BootLoader, BootLoader 启动蓝牙模块，手机APP通过蓝牙向网关写入主系统。

## BootLoader 功能

BootLoader 作为网关的基本系统，它在硬件出厂前就应该刷写完成。

1. 支持启动蓝牙模块，通过蓝牙与手机APP通讯，进行系统写入/更新操作。
2. 支持通过网络在线更新系统，云端向网关下发更新系统指令，网关下载新系统，向 BootLoader 写入更新标记，重启系统，BootLoader 接管，更新系统。

## 配置信息 (支持 蓝牙 写入/更新)

手机APP通过蓝牙向网关写入入网配置信息。

1. broker 地址
2. port 端口
3. protocol 协议
4. username 用户名
5. password 密码
6. farmID 农场ID
7. topic 主题
8. clientID 设备的唯一码，int32的纯数字
9. zoneKey 区 key
10. groupKey 组 key
11. nodeKey 网关 key
12. minPower 最小电量
13. maxPower 最大电量

config

```json
{
    "broker": "broker.alinode.com",
    "port": 8443,
    "protocol": "tcp",
    "username": "",
    "password": "",
    "farmID": 1,
    "topic": "farm/$farmID/action/#",
    "zoneKey": 0,
    "groupKey": 0,
    "nodeKey": 1
}
```

## 网关入网注册

网关通过入网配置信息，链接 MQTT 代理服务器，链接成功后发出注册消息。

订阅 config.topic 主题

## 基础数据类型

```go
$farmID uint64 // 农场ID
$zoneKey uint8 // 区域Key
$groupKey uint8 // 组Key
$nodeKey uint8 // 节点Key
$switchKey uint8 // 开关Key, 如果一个RTU支持4路控制，它的key值就为：1，2，3，4
$clientID uint32 // 设备的唯一码
```

## 基础指令

action

```json
{
    "ping": 1,
    "config": 2,
    "switch": 3,
    "task": 4,
    "clearTask": 5,
    "data": 6,
    "rotary": 7,
    "alert": 8,
}
```
### config 指令 (中优先级)

配置网关 

topic: farm/$farmID/action/$action.config/gateway/$clientID

payload:

```json
{
    "broker": "broker.alinode.com",
    "port": 8443,
    "protocol": "tcp",
    "username": "",
    "password": "",
    "farmID": 1,
    "topic": "farm/$farmID/action/#",
    "zoneKey": 0,
    "groupKey": 0,
    "nodeKey": 1
}
```

信息回传

topic：farm/$farmID/reply/$action.config/gateway/$clientID

成功：

```json
1
```

失败

```json
0
```

配置控制器

topic: farm/$farmID/action/$action.config/rtu/$clientID

payload:

```json
{
    "farmID": 1,
    "zoneKey": 0,
    "groupKey": 0,
    "nodeKey": 1,
    "type": "R330"
}
```

信息回传

topic：farm/$farmID/reply/$action.config/rtu/$clientID

成功：

```json
1
```

失败

```json
0
```

### ping 指令 (中优先级)

curPower 当前电量
curRSSI 当前信号强度

topic: farm/$farmID/action/$action.ping/zone/$zoneKey/group/$groupKey/node/$nodeKey


信息回传

topic：farm/$farmID/reply/$action.ping/zone/$zoneKey/group/$groupKey/node/$nodeKey

```json
{
    "curPower": $curPower,
    "curRSSI": $curRSSI,
    "switchKey": $switchKey,
    "ratio": $ratio
}
```

### switch 开关控制指令 (高优先级)

payload: 0 代表关闭, 100 代表全开

#### 控制 网关 控制器

$nodeKey = 1

topic: farm/$farmID/action/$action.switch/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

```json
100
```

信息回传 topic：farm/$farmID/reply/$action.switch/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

ratio: 0 - 100 代表已经执行到位的开度
ratio: 111 代表正在执行打开
ratio: 112 代表正在执行关闭

```json
100
```

#### 控制 RTU 控制器

$nodeKey > 1

topic: farm/$farmID/action/$action.switch/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

```json
100
```

信息回传 topic：farm/$farmID/reply/$action.switch/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

ratio: 0 - 100 代表已经执行到位的开度
ratio: 111 代表正在执行打开
ratio: 112 代表正在执行关闭

```json
100
```

### 轮灌状态上报 (高优先级)

topic: farm/$farmID/reply/$action.rotary/rotary/$rotaryID/group/$groupID

action: -1 终止 0 暂停 1 启动 

```json
{
    "action": 1,
    "startAt": $startAt,
    "doneAt": $doneAt
}
```

### 压力信息查询 (中优先级)

topic: farm/$farmID/action/$action.data/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

### 压力信息上报 (中优先级)

topic: farm/$farmID/reply/$action.data/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

```json
1.234
```

### 警报信息上报 (中优先级)

topic: farm/$farmID/reply/$action.alert/zone/$zoneKey/group/$groupKey/node/$nodeKey

value: 0 代表正常， 1 代表阀站倒下

```json
0
```

### task 任务控制指令 (低优先级)

topic: farm/$farmID/action/$action.task/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

1. 在 startAt 的时间，打开开关，10 秒后关闭开关

```json
{
    "startAt": 1648629056,
    "duration": 10
}
```

2. 在 startAt 的时间，打开开关，持续 60 秒，关闭开关，暂停 30 秒， 重复执行 20 次。

```json
{
    "startAt": 1648629056,
    "duration": 60,
    "repeatTimes": 20,
    "interval": 30
}
```

### clearTask 清除任务指令 (低优先级)

清除RTU控制器对应开关的任务

topic: farm/$farmID/action/$action.clearTask/zone/$zoneKey/group/$groupKey/node/$nodeKey/switch/$switchKey

## 电源管理

当网关是用电池供电的时候，电量小于 $config.minPower 的时候开始充电，充到电量 大于等于 $config.maxPower 停止充电。

如果采用多电池供电方案，需要支持对单独的每个 电池/电池组 进行管理，每个 电池/电池组 都可以独立工作。

## 出厂默认设置

1. 网关在出厂的时候默认 主题 topic 地址设置为 c/$clientID/#