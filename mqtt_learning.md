# Learn MQTT

MQTT是由IBM为物联网开发的一个通讯协议。

## 特点：
- 发布／订阅模式
- 基于TCP/IP
- Qos：
  * Qos 0: 至多一次
  * Qos 1: 至少一次
  * Qos 2: 刚好一次
- 消息简短

## 有用的参考
- [介绍](http://docs.emqtt.cn/zh_CN/latest/mqtt.html):了解发布／订阅模式，broker概念
- [github page](https://github.com/mqtt/mqtt.github.io/wiki/software?id=software)
- client lib：[mqtt.js](https://github.com/mqttjs/MQTT.js) or [others](https://github.com/mqtt/mqtt.github.io/wiki/libraries)
- Broker: 
  * [mosquito](https://mosquitto.org/):功能比较完全的broker，本次主要调研对象
  * [others](https://github.com/mqtt/mqtt.github.io/wiki/servers): 注意[compare the capabilities](https://github.com/mqtt/mqtt.github.io/wiki/server-support) of different brokers
  * [Public brokers](): useful for testing and prototyping


以下例子代码语言为typescript，用到的client lib为[mqtt.js](https://github.com/mqttjs/MQTT.js), broker为[mosquito](https://mosquitto.org/)

## 基本玩法
### 发布／订阅模式
1. 安装Mosquito (Mac OS), 并启动

```
brew install mosquitto
brew start services mosquitto
```
2. 新建ts项目，并引入mqtt client lib：[mqtt.js](https://github.com/mqttjs/MQTT.js)
```
npm install mqtt --save
```
3. example (为了简单，这里发布和订阅都是同一个客户端)
```javascript
import mqtt = require('mqtt')
let client = mqtt.connect('mqtt://127.0.0.1') //指定mqtt协议；其他可用协议：'mqtt', 'mqtts', 'tcp', 'tls', 'ws', 'wss'

client.on('connect', () => {
  client.subscribe('presence', {qos: 0})
  client.publish('presence', 'Hello mqtt', {qos: 2})
})
 
client.on('message', (topic, message) => {
  // message is Buffer
  console.log(message.toString())
  client.end()
})
```
### 客户端／服务器模式
可参考node模块：[mqtt-connnection](https://www.npmjs.com/package/mqtt-connection)

## MQTT特色花样
### QoS (Quality of Service)
- 整体的QoS取决于publisher和subcriber的QoS较小值。如publisherQoS=0，subcriberQoS=2，则整体QoS=0
- typescript example:
```javascript
  client.subscribe('presence', {qos: 0})
  client.publish('presence', 'Hello mqtt', {qos: 2})
```
### Retained Messages
publisher在发布消息时，可以设置此消息为retained，broker会将此消息发送给所有当前subcriber，并且`将此消息保留下来`，之后如果有新的subcriber加入，broker会把此消息发给它。

问题：broker如何将消息保留？有没有做持久化？

```javascript
let client1 = mqtt.connect('mqtt://127.0.0.1')

    client1.on('connect', () => {
        console.log("client1 connected to broker")
        client1.subscribe('presence', {qos: 2})
        client1.publish('presence', 'Hello mqtt', {qos: 2, retain: true})
        console.log("client1 published message")
    })
    
    client1.on('message', (topic, message) => {
        console.log("client1 received message:", message.toString())
        client1.end()
    })

    setTimeout(() => {
        let client2 = mqtt.connect('mqtt://127.0.0.1')
        client2.on('connect', () => {
            console.log("client2 connected to broker")
            client2.subscribe('presence', {qos: 2})
        })
        client2.on('message', (topic, message) => {
            console.log("client2 received message:", message.toString())
            client1.end()
        })
    }, 2000)

//output
client1 connected to broker
client1 published message
client1 received message: Hello mqtt
client2 connected to broker
client2 received message: Hello mqtt
```

### Clean Session
client连接broker的时候可以设置clean=false，如果client断开连接，broker会保留断线期间的QoS1和QoS2消息，并在client重新连接上来的时候发送给它。若要使用此功能，`连接的时候需要设置clientId`

```javascript
let client1 = mqtt.connect('mqtt://127.0.0.1', {clientId: 'client1', clean:false})
```
### Wills(遗愿)
client连接broker时，可以传递一个will message。当client异常退出时，broker会publish这个will message。Will message里面包含了topic，QoS，retain status这些信息

```javascript
let client1 = mqtt.connect('mqtt://127.0.0.1', { will:{
        topic: 'topic',
        payload: 'will send this message when interrupted',
        qos: 2,
        retain: true
    }})
```
### Bridge
### Web Socket