# ESP BLE Mesh 框架
本文首先对 ESP BLE Mesh 协议栈的主体框架进行了介绍，然后介绍了协议栈文件对应的实现与详细的协议栈接口，最后描述了与协议栈相关的任务(task)和一些可选的辅助程序。

## 协议栈框架图

![arch](images/arch.png)

ESP BLE MESH 协议栈主体由2大部分组成：`Mesh Networking`(黄色框图)，`Mesh Provisioning`(红色框图) 组成。
* Mesh Networking 负责 Node 以及 Provisioner 的网络消息处理等。
* Mesh Provisioning负责 Node 以及Provisioner 的启动配置流程。

应用层(蓝色框图)通过调用 API 和处理 Event 的方式和协议栈中的 `Mesh Networking` 与 `Mesh Provisioning` 进行交互。
ESP BLE Mesh协议栈建立在低功耗蓝牙技术之上,通过承载层使用低功耗蓝牙的广播以及连接通道进行数据交互，其中 承载层包括 `Advertising Bearer` 和 `GATT  Bearer` 。

ESP BLE Mesh协议栈采用的分层的方式设计的，数据包的处理会经过的层处理顺序是固定的，也就是数据包的处理过程会形成一个`消息流`(message flow)。`消息流`描述的是数据包经过了哪些层和层处理的先后顺序。从框架图上可以知道主要有4种类型的`消息流`,分别对应框架图的4列。
比如：`Bluetooth Low Energy Core` <--> `Advertising Bearer` <--> `Network Layer` <--> `Lower Transport Layer` <--> `Upper Transport Layer` <--> `Access Layer` <--> `Foundation Model Layer` <-->`Model Layer` <--> `API/Event` <--> `Application`。
其中每一层对数据包都会进行不同的处理：`Network Layer`会对网络数据包进行模糊化/去模糊化, 加密/解密等； `Lower Transport Layer`会对数据包进行分包和重组； `Upper Transport Layer`会对数据包进行应用层的加密解密等。

### Mesh Networking
 协议栈框架图中的 `Mesh Networking` 所包含的每一层的具体功能如下表所示：

| Layer     | Function |
| --------- | -------- |
| 模型（Model Layer）| 模型层与模型等的实施、以及诸如行为、消息、状态等的实施有关。|
| 基础模型（foundation model Layer）| 基础模型层负责实现与mesh网络配置和管理相关的模型。|
| 接入层（Access Layer）| 接入层主要负责应用数据的格式、定义并控制上层传输层对数据包的加密和解密等。|
| 上层传输层（upper transport Layer）| 负责对接入层进出的应用数据进行加密、解密和认证。它还负责称为“传输控制消息”（transport control messages）这一特殊的消息，包括与“friendship”相关的心跳和消息。 |
| 底层传输层（lower transport layer）| 在需要之时，底层传输层能够处理PDU的分段和重组。 |
| 网络层（network layer） | 网络层定义了各种消息地址类型和网络消息格式。中继和代理行为通过网络层实施。 |
| 广播承载层 (Advertising Bearer) | When using the advertising bearer, a mesh packet shall be sent in the Advertising Data of a `Bluetooth Low Energy advertising PDU` using the Mesh Message AD Type |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| GATT承载层 (GATT Bearer) | The GATT bearer uses the Proxy protocol to transmit and receive `Proxy PDUs` between two devices over a GATT connection |
| 代理服务 (Proxy Service) | The Mesh Proxy Service is used to enable a server to send and receive Proxy PDUs with a client. |

**Note：承载层（bearer layer）定义了如何使用底层低功耗协议传输Mesh 数据包。目前定义了两个承载层：广播承载层（Advertising Bearer）和 GATT 承载层。从协议栈框图中可以看出：GATT 承载层由 `GATT Bearer`，`Proxy Service`,`Proxy Protocol`组成，广播承载层由 Advertising Bearer 组成。**

### Mesh Provisioning
 协议栈框架图中的 `Mesh Provisioning` 每一层的具体功能如下表所示：

| Layer     | Function |
| --------- | -------  |
| 配置协议（Provisioning Protocol）| Provisioning is a process of adding an unprovisioned device to a mesh network, managed by a Provisioner. |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| PB-GATT  | PB-GATT is a provisioning bearer used to provision a device using Proxy PDUs to encapsulate Provisioning PDUs within the Mesh Provisioning Service . |
| PB-ADV  |PB-ADV is a provisioning bearer used to provision a device using Generic Provisioning PDUs over the advertising channels. |
| 配置服务 (Provisioning Service) | The Mesh Provisioning Service allows a Provisioning Client to provision a Provisioning Server to allow it to participate in the mesh network |
| GATT承载层 (GATT Bearer) | The GATT bearer uses the Proxy protocol to transmit and receive `Proxy PDUs` between two devices over a GATT connection |
| 广播承载层 (Advertising Bearer) | When using the advertising bearer, a mesh packet shall be sent in the Advertising Data of a `Bluetooth Low Energy advertising PDU` using the Mesh Message AD Type |

**代理协议 (Proxy Protocol)， GATT承载层 (GATT Bearer)和广播承载层 (Advertising Bearer)在协议栈的 Mesh Provisioning 和 Mesh Networking 中均可能使用。**

### 应用层
应用层通过调用 ESP BLE Mesh协议栈提供的API，以及处理协议栈上报的Event来实现相应的功能。

应用层（`Applications`）与 `API / Event`的交互：
* 应用层调用 API ：
应用层调用 API 主要进行如下操作：
  * 调用配网相关API进行配网操作。
  * 调用 model 相关的 API发送消息。
  * 调用设备属性相关的 API 获取设备的本地信息。
* 应用层处理 Event ：
应用层的设计方式是基于事件的，事件会携带参数给应用层。
事件主要分为两大类，调用 API 完成的事件以及 协议栈上报给应用层的事件，例如接收到节点消息等。
事件通过应用层注册的回调函数进行上报，同时回调函数中也会包含对事件的相应处理。

`API / Event` 与 ESP BLE MESH 协议栈的交互：
用户使用的 API 主要调用`Mesh Networking`，`Mesh Provisioning` 协议栈顶层提供的函数，也就是`Model Layer`,`Foundation Model Layer`,`Provisioning Protocol`提供的函数。`API / Event` 和协议栈的交互不会跨越协议栈的层进行操作。比如 API 不会调用到 `Network Layer` 相关的函数。

## 协议栈实现
协议栈代码在设计时主要用到了两个思想：分层思想和模块思想。
* 分层思想：从协议栈描述的层去设计文件，该类型的文件有一个明显的特征就是存在接口函数。其中 `Mesh Networking` 和 `Provisioning `都是基于该思想进行实现的。
* 模块思想：该文件实现一个独立的功能，供其它程序去调用。

### 协议栈接口图

![arch](images/interface.png)

接口图详细的描述了每一层对应的实现文件和文件的接口函数，也反应了数据包接收和发送的处理流程。

#### Mesh Networking 实现

| File | Functionality |
| ------ | ------ |
| `mesh_models/generic_client.c` | Send BLE Mesh Generic Client messages and receive corresponding response messages |
| `mesh_models/lighting_client.c` | Send BLE Mesh Lighting Client messages and receive corresponding response messages |
| `mesh_models/model_common.c` | BLE Mesh model related operations |
| `mesh_models/sensor_client.c` | Send BLE Mesh Sensor Client messages and receive corresponding response messages |
| `mesh_models/time_scene_client.c` | Send BLE Mesh Time Scene Client messages and receive corresponding response messages |
| `mesh_core/cfg_cli.c` | Send Configuration Client messages and receive corresponding response messages |
| `mesh_core/cfg_srv.c` | Receive Configuration Client messages and send proper response messages |
| `mesh_core/health_cli.c` | Send Health Client messages and receive corresponding response messages |
| `mesh_core/health_srv.c` | Receive Health Client messages and send proper response messages |
| `mesh_core/access.c` | BLE Mesh Access Layer |
| `mesh_core/transport.c` | BLE Mesh Lower/Upper Transport Layer |
| `mesh_core/net.c` | BLE Mesh Network Layer, IV Update, Key Refresh |
| `mesh_core/adv.c` | A task used to send BLE Mesh advertising packets and APIs used to allocate adv buffers |
| `mesh_core/mesh_bearer_adapt.c` | BLE Mesh Bearer Layer adapter，This file provides the interfaces used to receive and send BLE Mesh ADV & GATT related packets. |

**`mesh_bearer_adapt.c` 是协议栈框架图中的的 `Advertising Bearer`和`GATT  Bearer`的实现。**

#### Provisioning 实现
这部分代实现的时候考虑到 Node/Provisioner 的共存，将 Provisioning 部分拆分为两大块。
* `prov.c`,`proxy.c`,`beacon.c` 实现了节点（Node）端的配置行为。
* `provisioner_prov.c`,`provisioner_proxy.c`,`provisioner_beacon.c`,`provisioner_main.c`实现了 Provisioner 配置设备的功能。

| File | Functionality |
| ------ | ------ |
| `mesh_core/prov.c` | BLE Mesh Node provisioning (PB-ADV & PB-GATT) |
| `mesh_core/proxy.c` | BLE Mesh Node Proxy related functionalities |
| `mesh_core/beacon.c` | APIs used to handle BLE Mesh Beacons |

| File | Functionality |
| ------ | ------ |
| `mesh_core/provisioner_prov.c` | BLE Mesh Provisioner provisioning (PB-ADV & PB-GATT) |
| `mesh_core/provisioner_proxy.c` | BLE Mesh Provisioner Proxy related functionalities |
| `mesh_core/provisioner_beacon.c` | BLE Mesh Provisioner receives Unprovisioned Device Beacon and Secure Network Beacon |
| `mesh_core/provisioner_main.c` | BLE Mesh Provisioner manages networking inforamtion, e.g. provisioned nodes, local NetKeys, local AppKeys, etc. |


### 独立模块实现
采用独立模块的设计主要考虑到两个因素：
* 首先该模块不具备分层实现的特征，其次该模块能够完全独立起来，也就是该模块不需要依赖于其它模块的实现。
* 模块中的函数会被反复使用到，那么设计成模块是合理的。

| File | Functionality |
| ------ | ------ |
| `mesh_core/crypto.c` | Encrypt and decrypt BLE Mesh messages |
| `mesh_core/lpn.c` | BLE Mesh Low Power functionality |
| `mesh_core/friend.c` | BLE Mesh Friend functionality |
| `mesh_core/settings.c` | BLE Mesh Node NVS storage functionality |
| `mesh_core/mesh_main.c` | Initialize/enable/disable BLE Mesh |

## Other:
ESP BLE MESH 协议栈相关任务：
 *`btc_task`是一个任务，用于处理应用层和协议栈的请求(消息)。
 `btc_task`的工作原理为：应用层调用一个 API 或者底层上报 Event都会发送一个消息给 `btc_task`,`btc_task`在去调用应用层或协议栈之前注册的回调函数，调用回调函数的参数是消息中携带的。

 * `adv_task`: 此任务主要用于发送 mesh 的广播数据包。

辅助程序：
设计为用户可选的，不是协议栈的主体，但也十分重要。辅助程序的设计一般通过 menuconfig 的方式实现代码的裁剪。
* feature
	* friend ：实现朋友特性
	* lpn：实现低功耗特性
	* proxy：实现代理特性
	* relay：实现中继特性

* 网络管理相关程序
	* Mesh Network Creation procedure ：网络创建程序
	* Key Refresh procedure ：秘钥更新程序
	* IV Update procedure ：网络索引更新程序
	* IV Index Recovery procedure：网络索引恢复程序
	* Node Removal procedure：节点移除程序
