[TOC]

# ESP BLE Mesh 架构
ESP BLE Mesh 架构主要包含两大部分: ESP BLE Mesh 协议栈和 Mesh Models。
ESP BLE Mesh 协议栈是对 [Mesh Profile]() 的实现, Mesh Models 是对[Mesh_Model_Specification]()的实现。

本文将从ESP BLE Mesh 架构描述、ESP BLE Mesh 架构实现及 ESP BLE Mesh 辅助程序三个方面进行介绍。
* ESP BLE Mesh 架构描述
   * 描述了 ESP BLE Mesh 架构的 5 大部分，以及每个部分功能的简介。
* ESP BLE Mesh 架构实现
   * 从 ESP BLE Mesh 的文件的基本功能、文件与 ESP BLE Mesh 架构的对应关系、以及文件与文件调用的接口关系三个方面进行描述。
* ESP BLE Mesh 辅助程序
   * 描述了 ESP BLE Mesh 的辅助程序，比如 Mesh 网络管理，Mesh features 等。

## 1. ESP BLE Mesh 架构描述

ESP BLE Mesh 架构实现了 [Mesh Profile]() 和 [Mesh_Model_Specification]() 的所有功能，并通过了蓝牙官方[认证]()，ESP BLE Mesh 架构如图 1.1 所示：

![arch](images/arch.png)

						图 1.1 ESP BLE Mesh 架构图

ESP BLE Mesh 架构主要由 5 大部分组成:
* ESP BLE MESH 协议栈
	* `Mesh Networking` 负责 BLE Mesh 设备的网络消息处理等。
	* `Mesh Provisioning` 负责 BLE Mesh 设备的启动配置流程。
* `MESH Models`  实现了一些列的 Models ，比如：Generic Client models，Sensor Client models，Time Scene Client models and Lighting Cient models。
* `Mesh Bearers` 包括 `Advertising Bearer` 和 `GATT  Bearer`, ESP BLE Mesh 协议栈建立在低功耗蓝牙技术之上,通过承载层使用低功耗蓝牙的广播以及连接通道进行数据交互。
* `Applications` 是基于 ESP BLE Mesh 协议栈 和 `Mesh Models` 的应用程序，`Applications` 通过调用 API 和处理 Event 的方式和 ESP BLE Mesh 协议栈中的 `Mesh Networking` 与 `Mesh Provisioning` 进行交互，以及和 `Mesh Models` 提供的一系列 Model 进行交互。
* `Aux` 是 ESP BLE MESH 协议栈提供的辅助程序，每个辅助程序实现一个单独的功能，比如 Relay 功能等。

**Note: 代理协议 (Proxy Protocol)， GATT承载层 (GATT Bearer)和广播承载层 (Advertising Bearer)在协议栈的 Mesh Provisioning 和 Mesh Networking 中均可以使用。**

### 1.1 Mesh Networking

协议栈架构图中的 `Mesh Networking` 实现了如下
 * 实现了 Mesh 网络中的成员之间的通讯。
 * 实现了 Mesh 网络中的消息的加密解密。
 * 实现了 Mesh 网络资源（Network Key， IV Index, etc.）的管理。
 * 实现了 Mesh 消息的分包与重组。
 * 实现了 Mesh 节点的特性(Low Power, Friend, etc.)
 * 更多功能，请见[链接]()

 `Mesh Networking`功能的实现是基于层次结构的，每一层的功能如表 1.1 所示：

| Layer     | Function |
| --------- | -------- |
| 基础模型（Foundation model Layer）| 基础模型层负责实现与mesh网络配置和管理相关的模型。|
| 接入层（Access Layer）| 接入层主要负责应用数据的格式、定义并控制上层传输层对数据包的加密和解密等。|
| 上层传输层（upper transport Layer）| 负责对接入层进出的应用数据进行加密、解密和认证。它还负责称为“传输控制消息”（transport control messages）这一特殊的消息，包括与“friendship”相关的消息以及心跳消息。 |
| 底层传输层（lower transport layer）| 底层传输层能够处理 PDU 的分段和重组。 |
| 网络层（network layer） | 网络层定义了消息地址类型和网络消息格式，中继功能通过设备的网络层实施。 |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| 代理服务 (Proxy Service) | The Mesh Proxy Service is used to enable a server to send and receive Proxy PDUs with a client. |

					 表1.1  Mesh Networking 框架描述


### 1.2 Mesh Provisioning

协议栈架构图中的 `Mesh Provisioning` 实现了如下功能：
 * 实现了对 Mesh 设备的启动配置
 * 实现了 Mesh 网络资源的分配 (单播地址，IV Index，NetKey)。
 * 实现了对启动配置过程中 OOB 的支持
 * 更多功能，请见[链接]()
 
`Mesh Provisioning`功能的实现是基于层次结构的，每一层的具体功能如表 1.2 所示：

| Layer     | Function |
| --------- | -------  |
| 配置协议（Provisioning Protocol）| Provisioning is a process of adding an unprovisioned device to a mesh network, managed by a Provisioner. |
| PB-GATT  | PB-GATT is a provisioning bearer used to provision a device using Proxy PDUs to encapsulate Provisioning PDUs within the Mesh Provisioning Service . |
| PB-ADV  |PB-ADV is a provisioning bearer used to provision a device using Generic Provisioning PDUs over the advertising channels. |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| 配置服务 (Provisioning Service) | The Mesh Provisioning Service allows a Provisioning Client to provision a Provisioning Server to allow it to participate in the mesh network |

					表1.2  Mesh Provisioning 框架描述


### 1.3 Mesh Bearers
协议栈架构图中的 `Bearer` 实现了如下功能：
* 从 Bluetooth Low Energy Core 中获取数据传递给 ESP BLE MESH 协议栈。
* 从 ESP BLE MESH 协议栈 中获取数据传递给 Bluetooth Low Energy Core 。

`Bearer`可以理解为是基于 Bluetooth Low Energy Core 的承载层，为 ESP BLE MESH 协议栈实现接收和发送数据的功能。

| Layer     | Function |
| --------- | -------  |
| GATT承载层 (GATT Bearer) | The GATT bearer uses the Proxy protocol to transmit and receive `Proxy PDUs` between two devices over a GATT connection |
| 广播承载层 (Advertising Bearer) | When using the advertising bearer, a mesh packet shall be sent in the Advertising Data of a `Bluetooth Low Energy advertising PDU` using the Mesh Message AD Type |

					表1.3  Mesh Bearers 描述

### 1.4 Mesh Models
协议栈架构图中的 `Mesh Models` 实现了如下功能：
* 实现了 Generic Client models。
* 实现了 Sensor Client models。
* 实现了 Time and Scenes Client models。
* 实现了 lighting Client models。

| Layer     | Function |
| --------- | -------  |
| 模型层（Model Layer）| 模型层与模型等的实施、以及诸如行为、消息、状态等的实施有关。|

					表1.4  Mesh Models 描述

### 1.5 Mesh Applications

协议栈框架图中的 `Applications` 是通过调用 ESP BLE Mesh 协议栈提供的 API，以及处理协议栈上报的 Event 来实现相应的功能,有一些常见应用,比如网关，灯的应用程序等。


应用层（`Applications`）与 `API / Event` 的交互
* 应用层调用 API 

  * 调用配网相关API进行配网操作。
  * 调用 model 相关的 API发送消息。
  * 调用设备属性相关的 API 获取设备的本地信息。
* 应用层处理 Event 

 * 应用层的设计方式是基于事件的，事件会携带参数给应用层。
 * 事件主要分为两大类，调用 API 完成的事件以及协议栈上报给应用层的事件，例如接收到节点消息等。
    * 协议栈上报给应用层的事件又分为4大类
	   *  调用协议栈相关的 API 执行结束返回的 Event。
	   *  协议栈自动上报的 Event。
	   *  调用 Model 相关的 API 执行结束返回的 Event。
	   *  Model 主动上报的 Event。
 * 事件通过应用层注册的回调函数进行上报，同时回调函数中也会包含对事件的相应处理。

`API / Event` 与 ESP BLE MESH 协议栈的交互
* 用户使用的 API 主要调用`Mesh Networking`，`Mesh Provisioning` ，`Mesh models` 提供的函数。
* `API / Event` 和协议栈的交互不会跨越协议栈的层进行操作。比如 API 不会调用到 `Network Layer` 相关的函数。

## 2. ESP BLE Mesh 架构实现

ESP BLE Mesh 架构代码在设计时主要用到了两个思想：分层思想和模块思想。本章节根据设计思想进行分类，其中 2.1，2.2，2.3 章节使用的是分层实现方法，2.4 章节使用的是模块实现方法。

* **分层思想**：按照协议栈描述的层级去设计框架，该类型的文件有一个明显的特征就是存在接口函数。其中 `Mesh Networking` 和 `Mesh Provisioning `，`Mesh Bearers`都是基于该思想进行实现的，具体设计如图 2.1 所示

* **模块思想**：每个文件实现一个独立的功能，供其它程序去调用。

![arch](images/interface.png)

						图 2.1 协议栈接口图
					      
ESP BLE Mesh 架构是采用分层的方式进行设计的，数据包的处理会经过的层处理顺序是固定的，也就是数据包的处理过程会形成一个`消息流`,从协议栈接口图中我们可以清晰的看到从形成的一条条消息流。
协议栈接口图主要由 3 列组成，每一列中的文件与文件的数据交互以及所在的层次结构在图上均有具体的描述。

* `mesh_bearer_adapt.c`，`adv.c`，`net.c`，`transport.c`，`access.c`，`Foundation Models这一列对应了 Mesh Networking 的实现。

* `prov.c`, `proxy.c`,`beacon.c`这一列对应了 Mesh Provisioning 中的节点（Node）端的配置功能的实现。

* `provisioner_proxy.c`,`provisioner_prov.c`,`provisioner_beacon.c` 这一列对应了Mesh Provisioning 中的 Provisioner 的配置功能的实现。


**Note：用户可以根据协议栈接口图中反应的关系去分析代码。`Mesh Models`对应图 2.1 中的 `Application Model` 部分**

### 2.1 Mesh Networking 实现

`Mesh Networking` 中相关文件与每个文件实现的功能如表 2.1 所示：

| File | Functionality |
| ------ | ------ |
| `mesh_core/cfg_cli.c` | Send Configuration Client messages and receive corresponding response messages |
| `mesh_core/cfg_srv.c` | Receive Configuration Client messages and send proper response messages |
| `mesh_core/health_cli.c` | Send Health Client messages and receive corresponding response messages |
| `mesh_core/health_srv.c` | Receive Health Client messages and send proper response messages |
| `mesh_core/access.c` | BLE Mesh Access Layer |
| `mesh_core/transport.c` | BLE Mesh Lower/Upper Transport Layer |
| `mesh_core/net.c` | BLE Mesh Network Layer, IV Update, Key Refresh |
| `mesh_core/adv.c` | A task used to send BLE Mesh advertising packets, a callback used to handle received advertising packets and APIs used to allocate adv buffers |

					表2.1  Mesh Networking 文件描述

### 2.2 Mesh Provisioning 实现

这部分代实现的时候考虑到 Node/Provisioner 的共存，将 Provisioning 部分拆分为两大块。
* `prov.c`,`proxy.c`,`beacon.c` 实现了节点（Node）端的配置行为，如表 2.2 所示：

| File | Functionality |
| ------ | ------ |
| `mesh_core/prov.c` | BLE Mesh Node provisioning (PB-ADV & PB-GATT) |
| `mesh_core/proxy.c` | BLE Mesh Node Proxy related functionalities |
| `mesh_core/beacon.c` | APIs used to handle BLE Mesh Beacons |

					表2.2  Mesh Provisioning (Node) 文件描述

*`provisioner_prov.c`,`provisioner_proxy.c`,`provisioner_beacon.c`实现了 Provisioner 配置设备的功能，如表 2.3 所示：

| File | Functionality |
| ------ | ------ |
| `mesh_core/provisioner_prov.c` | BLE Mesh Provisioner provisioning (PB-ADV & PB-GATT) |
| `mesh_core/provisioner_proxy.c` | BLE Mesh Provisioner Proxy related functionalities |
| `mesh_core/provisioner_beacon.c` | BLE Mesh Provisioner receives Unprovisioned Device Beacon and Secure Network Beacon |


					表2.3  Mesh Provisioning(Provisioner)文件描述
### 2.3 Mesh Bearers 实现

| File | Functionality |
| ------ | ------ |
| `mesh_core/mesh_bearer_adapt.c` | BLE Mesh Bearer Layer adapter，This file provides the interfaces used to receive and send BLE Mesh ADV & GATT related packets. |
	
					表2.4  Mesh Bearers 文件描述
					
**`mesh_bearer_adapt.c` 是 Mesh Networking 框架中 `Advertising Bearer`和`GATT  Bearer`的实现。**

### 2.4 Mesh Models 实现

| File | Functionality |
| ------ | ------ |
| `mesh_models/generic_client.c` | Send BLE Mesh Generic Client messages and receive corresponding response messages |
| `mesh_models/lighting_client.c` | Send BLE Mesh Lighting Client messages and receive corresponding response messages |
| `mesh_models/model_common.c` | BLE Mesh model related operations |
| `mesh_models/sensor_client.c` | Send BLE Mesh Sensor Client messages and receive corresponding response messages |
| `mesh_models/time_scene_client.c` | Send BLE Mesh Time Scene Client messages and receive corresponding response messages |

					表2.5  Mesh Models 文件描述
### 2.5 Mesh Applications 实现
我们已经为客户开发提供了一些列的 Demo，用户可以使用 Demo 进行产品开发。
* [ble_mesh_client_model]()
* [ble_mesh_console]()
* [ble_mesh_fast_provision]()
* [ble_mesh_client_model]()
* [ble_mesh_node]()
* [ble_mesh_provisioner]()
* [ble_mesh_wifi_coexist]()


## 3. 辅助程序:

辅助程序指的是 ESP BLE Mesh 协议栈中可选的功能。辅助程序的设计一般通过 menuconfig 的方式实现代码的裁剪。[menuconfig 链接]()
### Feature
* friend ：实现朋友特性
* lpn：实现低功耗特性
* proxy：实现代理特性
* relay：实现中继特性

### Network Management
* Mesh Network Creation procedure ：网络创建程序
* Key Refresh procedure ：秘钥更新程序
* IV Update procedure ：网络索引更新程序
* IV Index Recovery procedure：网络索引恢复程序
* Node Removal procedure：节点移除程序

### Mesh module

采用独立模块的设计主要考虑到两个因素：
* 首先该模块不具备分层实现的特征，其次该模块能够完全独立起来，也就是该模块不需要依赖于其它模块的实现。
* 模块中的函数会被反复使用到，那么设计成模块是合理的。独立模块如表 3.1 所示：

| File | Functionality |
| ------ | ------ |
| `mesh_core/crypto.c` | Encrypt and decrypt BLE Mesh messages |
| `mesh_core/lpn.c` | BLE Mesh Low Power functionality |
| `mesh_core/friend.c` | BLE Mesh Friend functionality |
| `mesh_core/settings.c` | BLE Mesh Node NVS storage functionality |
| `mesh_core/mesh_main.c` | Initialize/enable/disable BLE Mesh |
| `mesh_core/provisioner_main.c` | BLE Mesh Provisioner manages networking inforamtion, e.g. provisioned nodes, local NetKeys, local AppKeys, etc. |

					表3.1  模块文件描述




BLE Mesh Support
* [*]   Support Duplicate Scan in BLE Mesh
* [*]   Enable BLE Mesh Fast Provisioning
* -*-   Support for BLE Mesh Node
* -*-   Support for BLE Mesh Provisioner
  * (20)    Maximum number of unprovisioned devices that can be added to device queue
  * (20)    Maximum number of nodes whose information can be stored
  * (20)    Maximum number of devices that can be provisioned by provisioner
  * (10)    Maximum number of PB-ADV running at the same time by provisioner
  * (4)     Maximum number of PB-GATT running at the same time by provisioner
  * (3)     Maximum number of mesh subnets that can be created by provisioner
  * (9)     Maximum number of application keys that can be owned by provisioner
* -*-   BLE Mesh Provisioning support
* -*-   Provisioning support using the advertising bearer (PB-ADV)
* [*]   Provisioning support using GATT (PB-GATT)
* -*-   BLE Mesh Proxy protocol support
* [*]   BLE Mesh GATT Proxy Service
  *  (60)    Node Identity advertising timeout
  *  (1)   Maximum number of filter entries per Proxy Client
*  [*]   BLE Mesh net buffer pool usage tracking
*  [*]   Store BLE Mesh Node configuration persistently
 *  (2)     Delay (in seconds) before storing anything persistently 
 * (128)   How often the sequence number gets updated in storage 
 * (5)     Minimum frequency that the RPL gets updated in storage 
* (1)   Maximum number of mesh subnets per network
* (1)   Maximum number of application keys per network
* (1)   Maximum number of application keys per model
* (1)   Maximum number of group address subscriptions per model
* (1)   Maximum number of Label UUIDs used for Virtual Addresses
* (10)  Maximum capacity of the replay protection list
* (10)  Network message cache size
* (20)  Number of advertising buffers
* (6)   Maximum number of simultaneous outgoing segmented messages
* (1)   Maximum number of simultaneous incoming segmented messages
* (384) Maximum incoming Upper Transport Access PDU length
* [*]   Relay support
* [*]   Support for Low Power features
 * [*]     Perform Friendship establishment using low power
 * [*]     Automatically start looking for Friend nodes once provisioned
 * (15)      Time from last received message before going to LPN mode
 * (8)     Retry timeout for Friend requests
 * (0)     RSSIFactor, used in Friend Offer Delay calculation
 * (0)     ReceiveWindowFactor, used in Friend Offer Delay calculation
 * (1)     Minimum size of the acceptable friend queue (MinQueueSizeLog)
 * (100)   Receive delay requested by the local node
 * (300)   The value of the PollTimeout timer
 * (300)   The starting value of the PollTimeout timer
 * (10)    Latency for enabling scanning
 * (8)     Number of groups the LPN can subscribe to
* [*]   Support for acting as a Friend Node
 * (255)   Friend Receive Window 
 * (16)    Minimum number of buffers supported per Friend Queue 
 * (3)     Friend Subscription List Size 
 * (2)     Number of supported LPN nodes 
 * (1)     Number of incomplete segment lists per LPN 
* [*]   Disable BLE Mesh debug logs (minimize bin size)
* [*]   Used the IRQ lock instead of task lock
* (4000) Timeout(ms) for client message response
* Support for BLE Mesh Client Models  --->
* [*]   Test the IV Update Procedure
