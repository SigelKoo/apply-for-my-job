### MAC地址

MAC（Media Access Control Address），直译为媒体存取控制位址，也称为局域网地址（LAN Address），以太网地址（Ethernet Address）或物理地址（Physical Address）

- 用来确认网络设备位置的地址。

- 每一个设备都有唯一的MAC地址。

- MAC地址共48位，使用十六进制表示。

### 以太网

Ethernet

- 以太网是一种使用广泛的局域网技术

- 以太网是一种应用于数据链路层的协议

- 使用以太网可以完成相邻设备的数据帧传输

以太网是一种广播类型的网络，而PPP网是点对点的，当然还有其他类型的网络

以太网是基于数据包交换的，而传统电话网ISDN/PSTN是基于电路交换的

以太网是一种局域网技术，而已经被淘汰的拨号上网（ADSL）、十年前左右的宽度上网/猫（PPPoE）、现在的光纤上网/光猫（EPON和GPON）是广域网技术

PPPoE和EPON中的E就是Ethernet，原本是局域网设计的技术，与其他技术结合后可为广域网服务

Wifi中也有以太网的身影

#### 以太网的格式

```
目的地址|源地址|类型|帧数据|CRC
```

目的地址和源地址为MAC地址，CRC是循环冗余校验码

#### 相邻设备数据传输

1. E知道地址的情况：
![img](https://img-blog.csdnimg.cn/201911201818248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FsaXZlbjE=,size_16,color_FFFFFF,t_70)

2. E不知道地址的情况：E广播并记录地址后再传输帧数据

![img](https://img-blog.csdnimg.cn/2019112018211624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FsaXZlbjE=,size_16,color_FFFFFF,t_70)









