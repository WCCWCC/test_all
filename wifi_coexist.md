# 1. Introduction
## 1.1 Demo Function
This is a demo that wifi and bluetooth coexist.

Testing the transfer rate of wifi when Bluetooth is working properly.

The function of Bluetooth is the function of ble_mesh_fast_prov_server.

在ble_mesh_fast_prov_server例程的基础之上，添加了wifi的基本功能。
wifi通过命令行的方式，实现了哪些wifi的命令

重点是共存：1蓝牙功能  2wifi功能

# 2. How to use this demo
//在wifi高速传输的情况下，使用快速配网的功能。

Interval表示时间间隔,Bandwidth是时间间隔里的传输速率,TCP server , TCP  client.
//将关键的log粘贴出来

# menuconfig 选项说明


# 3. 添加新的wifi命令
//这里需要将wifi的api粘贴出来


# 4. 扩展蓝牙的功能。
// 添加model
