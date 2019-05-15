# ESP BLE MESH 框架
本文首先对 ESP BLE MESH 协议栈进行了简单的介绍，然后介绍了协议栈的文件的基本功能，接下来重点分析了协议栈的主体框架，最后描述了协议栈的详细的接口图。

## 协议栈框架图

![arch](images/arch.png)

ESP BLE MESH 协议栈主体由2大部分组成：`Mesh Networking`(黄色框图)，`Provisioning`(红色框图) 组成。
* Mesh Networking 负责设备入网后的消息处理。
* Provisioning 负责设备入网前的消息处理。

应用层(蓝色框图)通过调用 API 和处理 Event 的方式和协议栈中的 `Mesh Networking` 与 `Provisioning` 进行交互。
ESP BLE MESH 协议栈建立在低功耗蓝牙技术之上,通过 `Adapt Layer`（适配层） 和 `Bluetooth Low Energy Core` 进行交互。其中 `Adapt Layer`由 `Advertising Bearer` 和 `GATT  Bearer` 组成。

ESP BLE MESH 协议栈采用的分层的方式设计的，数据包的处理会经过的层处理顺序是固定的，也就是数据包的处理过程会形成一个`处理流`。`处理流`描述的是数据包经过了哪些层与层处理的先后顺序。从框架图上可以知道主要有4种类型的处理流。
比如：`Bluetooth Low Energy Core` <--> `Advertising Bearer` <--> `Network Layer` <--> `Lower Transport Layer` <--> `Upper Transport Layer` <--> `Access Layer` <--> `Foundation Model Layer` <-->`Model Layer` <--> `API/Event` <--> `Aplication`。
其中每一层对数据包都会进行不同得到处理：`Network Layer`会对数据包进行网络层的加密解密； `Lower Transport Layer`会对数据包进行分包和重组； `Upper Transport Layer`会对数据包进行应用层的加密解密等。

### Mesh Networking (框架图上面有的都要进行描述)
  蓝牙mesh网络引入了全新的协议栈,这一协议栈建立在低功耗蓝牙技术之上，该协议栈分为7层，其每层所具有的功能如下表所示：

| Layer     | Function |
| --------- | -------- |
| 模型（models）    | 模型层与模型等的实施、以及诸如行为、消息、状态等的实施有关。|
| 接入层（access layer）     | 负责应用数据的格式、定义并控制上层传输层中执行的加密和解密过程，并在将数据转发到协议栈之前，验证接收到的数据是否适用于正确的网络和应用。|
| 基础模型（foundation models）| 基础模型层负责实现与mesh网络配置和管理相关的模型。|
| 上层传输层（upper transport layer）| 负责对接入层进出的应用数据进行加密、解密和认证。它还负责称为“传输控制消息”（transport control messages）这一特殊的消息，包括与“friendship”相关的心跳和消息。 |
| 底层传输层（lower transport layer）| 在需要之时，底层传输层能够处理PDU的分段和重组。 |
| 网络层（network layer） | 网络层定义了各种消息地址类型和网络消息格式。中继和代理行为通过网络层实施。 |
| 广播承载层 (Advertising Bearer) | 广播通道 |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| GATT承载层 (GATT Bearer) | 非广播通道 |
| 代理服务 (Proxy Service) | The Mesh Proxy Service is used to enable a server to send and receive Proxy PDUs with a client. |

**承载层（bearer layer）定义了如何使用底层低功耗堆栈传输PDU。目前定义了两个承载层：广播承载层（Advertising Bearer）和 GATT 承载层。其中 GATT 承载层由 `GATT Bearer`，`Proxy Service`,`Proxy Protocol`组成。**

###  
配置协议是用来将未配置设备加入到mesh网络中的，该协议分为3层，每层的功能如下表所示：

| Layer     | Function |
| --------- | -------  |
| 配置协议（Provisioning Protocol）| 实现对设备进行配置的功能 |
| 代理协议 (Proxy Protocol)     | The proxy protocol enables nodes to send and receive Network PDUs, mesh beacons, proxy configuration messages and Provisioning PDUs over a connection-oriented bearer. |
| PB-GATT  | PB-GATT is a provisioning bearer used to provision a device using Proxy PDUs to encapsulate Provisioning PDUs (see Section 5.4) within the Mesh Provisioning Service . |
| PB-ADV  |PB-ADV is a provisioning bearer used to provision a device using Generic Provisioning PDUs over the advertising channels. |
| 配置服务 (Provisioning Service) | The Mesh Provisioning Service allows a Provisioning Client to provision a Provisioning Server to allow it to participate in the mesh network |
| GATT承载层 (GATT Bearer) | 非广播通道 |
| 广播承载层 (Advertising Bearer) | 广播通道 |

### 应用层

* 应用层处理event ：
应用层的设计方式是基于事件的，事件会携带参数给应用层。
事件主要分为两大类，调用 API 完成事件和 ESP BLE MESH 协议栈上报给用户的事件。
事件的处理是应用层向协议栈注册回调函数，回调函数中会实现对应的事件的处理程序。
* 应用层调用API ：
用户调用 API 主要进行3大类进行操作：
  * 在配网过程中的 API，进行配网操作。
  * 使用 Model 进行通讯的 API，通讯是指的是通过 Model 在节点与节点之间收发消息。
  * 使用 API 设置本地变量，也就是设置节点自己拥有的数据结构。

用户使用的 API 主要调用`Mesh Networking`，`Provisioning` 协议栈顶层提供的函数，也就是`Model Layer`,`Foundation Model Layer`,`Provisioning Protocol`提供的函数。API和协议栈的交互不会跨越协议栈的层进行操作。比如 API 不会调用到 `Network Layer` 相关的函数。


## 代码结构介绍
协议栈代码文件主要从两个角度去实现的：分层思想和模块思想。
* 分层思想：从协议栈描述的层去设计文件，该类型的文件有一个明显的特征就是存在接口函数。
* 模块思想：该文件实现一个独立的功能，供其它程序去调用。

### 协议栈层次结构

![arch](images/interface.png)

接口图详细的描述了每一层对应的实现文件和文件的接口函数，也反应了数据包接收和发送的处理流程。
每个源文件的功能如下表所示：
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

#### Provisioning 实现
这部分代实现的时候考虑到 Node/Provisioner 的共存，

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
| `mesh_core/mesh_main.c` | Initialize/enable/disable BLE Mesh |


## 独立模块

friend + lpn ++

| File | Functionality |
| ------ | ------ |
| `mesh_core/crypto.c` | Encrypt and decrypt BLE Mesh messages |
| `mesh_core/lpn.c` | BLE Mesh Low Power functionality |
| `mesh_core/friend.c` | BLE Mesh Friend functionality |
| `mesh_core/settings.c` | BLE Mesh Node NVS storage functionality |


## Other:
ESP BLE MESH 协议栈相关任务：
 * `btc_task`: 应用层和协议栈之间的接口设计思想，应用层调用一个 API 或者底层上报 Event都会发送一个消息给 `btc_task`,`btc_task`在去调用应用层或协议栈之前注册的回调函数，调用回调函数的参数是消息中携带的。`btc_task`是一个任务，用于处理应用层和协议栈的请求(消息)。

 * `adv_task`: 此任务主要用于发送 mesh 的广播数据包。

辅助程序设计为用户可选的，不是协议栈的主体，但也十分重要。辅助程序的设计一般通过 menuconfig 的方式实现代码的动态裁剪。
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
