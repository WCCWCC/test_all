# 1. Introduction
## 1.1 Demo Function
[here](https://github.com/WCCWCC/test_all/blob/master/EspBleMesh.md)

## 1.2 Node Composition
This demo has only one element, in which the following three models are implemented:
- **Configuration Server model**: The role of this model is mainly to configure Provisioner device’s AppKey and set up its relay function, TTL size, subscription, etc.
- **Configuration Client model**: This model is used to represent an element that can control and monitor the configuration of a node.
- **Generic OnOff Server model**: This model implements the node's onoff state.

- **Vender Server model**:This model implements the node's `fast_prov_server` state.
- **Vender Client model**:This model is used to control the `fast_prov_server` state.


## 2. Code Analysis
### 2.1  Data Structure
`example_fast_prov_server_t` fast_prov_server. 
I will classify the data structure and introduce the behavior of the variables.

### 2.1.1 Provider role and state
Different provisers have different behaviors，It’s helpful to understand the role concept to understand the code.

1. Phone - Top Provisioner
2. A device been provisioned by Phone - Primary Provisioner
3. Devices been provisioned and changed to provisioner - Temporary Provisioner
4. Devices been provisioned and not changed to provisioner  - Node

| variable name        |Description               |
| ---------------------|------------------------- |
| `primary_role`      | proviser identity |
| `state`      | Fast prov state -> 0: idle, 1: pend, 2: active |
| `srv_flags`  | flag to `DISABLE_FAST_PROV_START`,`SEND_ALL_NODE_ADDR_START`,`RELAY_PROXY_DISABLED`,`SRV_MAX_FLAGS`|


### 2.1.2 Provider Address management
1. Assign a unicast address to prevent address conflicts.
2. The way the address is allocated is the average allocation method.
3. Each proviser has an address range and a maximum provisioning number.Each provisioning device is assigned a subset of its address range.

example: A configurator's address range is 0 to 100 and  a maximum provisioning number is 5.The first node to be networked is assigned an address range of 0 to 20.The second node to be networked is assigned an address range of 20 to 40.

| variable name        |Description               |
| ----------------------|------------------------- |
| `unicast_min`      | Minimum unicast address can be send to other nodes |
| `unicast_max`      | Maximum unicast address can be send to other nodes |
| `unicast_cur`      | Current unicast address can be assigned|
| `unicast_step`     | Unicast address change step|

### 2.1.3 Provider's cached data
1. Cached data is used to initialize proviser.
2. Call `esp_ble_mesh_set_fast_prov_info` API and `esp_ble_mesh_set_fast_prov_action` API.
3. Then This node will have the ability to provisioning other unprovisioned device .

| variable name        |Description               |
| ----------------------|------------------------- |
| `flags`       |Flags state|
| `iv_index`    |Iv_index state|
| `net_idx`     |Netkey index state |
| `group_addr`  |Subscribed group address |
| `prim_prov_addr`  |Unicast address of Primary Provisioner |
| `match_val[16]`  |Match value to be compared with unprovisioned device UUID |
| `match_len`   | Length of match value to be compared |
| `max_node_num`   | The maximum number of devices can be provisioned by the Provisioner |
| `prov_node_cnt`   | Number of self-provisioned nodes |
| `app_idx`   |  AppKey index of the application key added by other Provisioner |
| `top_address`   | Address of the device(e.g. phone) which triggers fast provisioning |


### 2.1.4 Provider's Timer
`send_all_node_addr_timer` is used to collect the addresses of all nodes.
if temporary provisioner provisioning unprovisioned device,the timer will restart or start.
if timer timeout,temporary provisioner will send massage(Address information) to primary provisioner.

`disable_fast_prov_timer` is used to turn off the ability to provisioning.
If the app controls the node, the node will start the timer.In order to turn off the provisioning capabilities of all nodes.You need to broadcast the way to control the node.Then all node will disable provisioning function.

| variable name        |Description               |
| ----------------------|------------------------- |
| `disable_fast_prov_timer`       |Used to disable fast provisioning|
| `send_all_node_addr_timer`      |Used to send all node addresses to top provisioner|

### 2.2  model definition

#### 2.2.1 Vender Server model definition
1. `fast_prov_server` has been introduced above.
2. `fast_prov_srv_op` used to register minimum length of the message.The message is identity by opcode.If the length of the message is less than the minimum value, the bottom layer will be directly discarded.
3. `example_fast_prov_server_init` Registered callback function with timer timeout and initialize the model-related variables in the data structure.
4. `fast_prov_server` is the vender server's states.
5. `CID_ESP` and `ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_SRV` is used to identity the vender server model.The vender server model's model id is `ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_SRV`.
```c
example_fast_prov_server_t fast_prov_server = {
    .primary_role  = false,
    .max_node_num  = 6,
    .prov_node_cnt = 0x0,
    .unicast_min   = ESP_BLE_MESH_ADDR_UNASSIGNED,
    .unicast_max   = ESP_BLE_MESH_ADDR_UNASSIGNED,
    .unicast_cur   = ESP_BLE_MESH_ADDR_UNASSIGNED,
    .unicast_step  = 0x0,
    .flags         = 0x0,
    .iv_index      = 0x0,
    .net_idx       = ESP_BLE_MESH_KEY_UNUSED,
    .app_idx       = ESP_BLE_MESH_KEY_UNUSED,
    .group_addr    = ESP_BLE_MESH_ADDR_UNASSIGNED,
    .prim_prov_addr = ESP_BLE_MESH_ADDR_UNASSIGNED,
    .match_len     = 0x0,
    .pend_act      = FAST_PROV_ACT_NONE,
    .state         = STATE_IDLE,
};

static esp_ble_mesh_model_op_t fast_prov_srv_op[] = {
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET,      3,  NULL },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_ADD,   16, NULL },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR,     2,  NULL },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET, 0,  NULL },
    ESP_BLE_MESH_MODEL_OP_END,
};

static esp_ble_mesh_model_t vnd_models[] = {
    ESP_BLE_MESH_VENDOR_MODEL(CID_ESP, ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_SRV,
    fast_prov_srv_op, NULL, &fast_prov_server),
};
static esp_ble_mesh_elem_t elements[] = {
    ESP_BLE_MESH_ELEMENT(0, root_models, vnd_models),
};
//
err = example_fast_prov_server_init(&vnd_models[0]);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to initialize fast prov server model", __func__);
    return err;  
}
```
#### 2.2.2 Vender Client model definition
1. `fast_prov_cli_op_pair` used to register corresponding message response.
```c
static const esp_ble_mesh_client_op_pair_t fast_prov_cli_op_pair[] = {
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET,      ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS      },
};
```
example:vender client model send massage with opcode `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET`,then the vender server model will respond massage with opcode `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS`.If no corresponding response is received,the vender server model will receive timeout.

if you don't want receive response massgae,you should written as follows。
```c
static const esp_ble_mesh_client_op_pair_t fast_prov_cli_op_pair[] = {
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET,      NULL      },
};
```
2. `example_fast_prov_server_init` Registered callback function with timer timeout and initialize the model-related variables in the data structure.
3. `CID_ESP` and `ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_CLI` is used to identity the vender server model.The vender client model's model id is `ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_CLI`.
```c
static const esp_ble_mesh_client_op_pair_t fast_prov_cli_op_pair[] = {
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET,      ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS      },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_ADD,   ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_STATUS   },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR,     ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_ACK    },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET, ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_STATUS },
};
esp_ble_mesh_client_t fast_prov_client = {
    .op_pair_size = ARRAY_SIZE(fast_prov_cli_op_pair),
    .op_pair = fast_prov_cli_op_pair,
};

static esp_ble_mesh_model_op_t fast_prov_cli_op[] = {
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS,    1, NULL },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_STATUS, 2, NULL },
    { ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_ACK,  0, NULL },
    ESP_BLE_MESH_MODEL_OP_END,
};

static esp_ble_mesh_model_t vnd_models[] = {
    ESP_BLE_MESH_VENDOR_MODEL(CID_ESP, ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_CLI,
    fast_prov_cli_op, NULL, &fast_prov_client),
};
static esp_ble_mesh_elem_t elements[] = {
    ESP_BLE_MESH_ELEMENT(0, root_models, vnd_models),
};
err = esp_ble_mesh_client_model_init(&vnd_models[1]);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to initialize fast prov client model", __func__);
    return err;
}
```

## 2.3 Message opcode
"Opcode-send"Represents the message that the client sends to the server.

"Opcode-ack" Represents the message that the server sent to the client.

|meaning | Opcode-send   | Opcode-ack   |                  
| -----| ------------- | -------------|
//INFO_SET
|opcode：| `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET` | `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS`    |
|function：| This message contains all the information as a Provisioner |Check each field of the Provisioner information and set the corresponding flag bit. The returned status is variable.|
|parameters：|structfast_prov_info_set|status_bit_mask,status_ctx_flag,status_unicast,status_net_idx,status_group,status_pri_prov,status_match,status_action|
//NODE_ADDR
|opcode：| `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR`  | `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_ACK`  |
|function：|  Temporary Provisioner reports the address of the node it is provisered. |Used to check if the message was sent successfully. |
|parameters：|Address array    |No parameters   |
//ADDR_GET
|opcode：| `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET` | `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_STATUS`  |
|function：|Top Provisioner gets the address of all nodes obtained from Primary Provisioner. | Returns the address of all nodes, but does not contain its own.     |
|parameters：|No parameters    |Address array    |
//NET_KEY_ADD
|opcode：| `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_ADD`   | `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_STATUS`    |
|parameters：| not use   |not use     |

### 2.4 model callback
#### 2.4.1 Vender server model callback
```c
    esp_ble_mesh_register_custom_model_callback(example_ble_mesh_custom_model_cb);
    esp_ble_mesh_register_prov_callback(example_ble_mesh_provisioning_cb);
```
1. Callback trigger.
>Receiving a message about onoff client mode will trigger this callback function
>call send messgae API about onoff client model.

2. The event that the callback function needs to handle.
I will sort by different mdoel.

| Event name    | Opcode      |Description                                 |
| ------------- | ------------|------------------------------------------- |
//onoff server model
| ESP_BLE_MESH_MODEL_OPERATION_EVT|ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET | receive `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` message |
| ESP_BLE_MESH_MODEL_OPERATION_EVT|ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK|received `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK` message  |
//vender server model
| ESP_BLE_MESH_MODEL_OPERATION_EVT | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET  | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET` message    |
| ESP_BLE_MESH_MODEL_OPERATION_EVT | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR    | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR` message |
| ESP_BLE_MESH_MODEL_OPERATION_EVT | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET    | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET` message  |
//proviser 
|ESP_BLE_MESH_SET_FAST_PROV_INFO_COMP_EVT|not opcode| call API `esp_ble_mesh_set_fast_prov_info` will trigger the event  |
|ESP_BLE_MESH_SET_FAST_PROV_ACTION_COMP_EVT|not opcode| call API `esp_ble_mesh_set_fast_prov_action` will trigger the event   |
//onoff client
|ESP_BLE_MESH_CFG_CLIENT_SET_STATE_EVT|ESP_BLE_MESH_MODEL_OP_APP_KEY_ADD|会调用API发送info set消息 |
|ESP_BLE_MESH_CFG_CLIENT_TIMEOUT_EVT|ESP_BLE_MESH_MODEL_OP_APP_KEY_ADD|超时事件|

#### 2.4.2 Vender client model callback
```c
    esp_ble_mesh_register_custom_model_callback(example_ble_mesh_custom_model_cb);
```
1. Callback trigger.
>Receiving a message about onoff server mode will trigger this callback function
>call send messgae API about onoff server model.
2. The event that the callback function needs to handle.

| Event name    | Opcode      |Description                                 |
| ------------- | ------------|------------------------------------------- |
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS  | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS` message|
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_STATUS | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NET_KEY_STATUS` message|
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_ACK  | receive `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_ACK` message |
| ESP_BLE_MESH_CLIENT_MODEL_SEND_TIMEOUT_EVT     | client_send_timeout.opcode    | send message timeimeout event|

### 2.5 vender model send messgae
#### 2.5.1 vender client send messgae
`esp_ble_mesh_client_model_send_msg` This API used to vender client model send messge.

| parameter name        |Description               |
| ----------------------|------------------------- |
| `model`       | Pointer to the client model structure  |
| `ctx.net_idx` | NetKey Index of the subnet through which to send the message. |
| `ctx.app_idx` | AppKey Index for message encryption. |
| `ctx.addr`    | Remote address/Destination address |
| `ctx.send_ttl`| message TTL,relay parameter |
| `ctx.send_rel`| Force sending reliably,Waiting for a response from the message   |
| `opcode`      | The Message opcode  |
| `msg->len`    | Length of the  `msg->data`|
| `msg->data`   | pointer to send data|
| `msg_timeout` | Time to wait for a Acknowledgement，The default is 4000ms  |
|`true`         | Set whether the message needs Acknowledgement |
| `msg_role`    | message role (node/proviser)  |

```c
esp_ble_mesh_msg_ctx_t ctx = {
    .net_idx  = info->net_idx,
    .app_idx  = info->app_idx,
    .addr     = info->dst, 
    .send_rel = false,
    .send_ttl = 0,
 };
 err = esp_ble_mesh_client_model_send_msg(model, &ctx,
        ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET,
        msg->len, msg->data, info->timeout, true, info->role);
```

#### 2.5.2 vender server send messgae

`esp_ble_mesh_server_model_send_msg` Model needs to bind appkey before sending a message.The send function of the vender server model and the sig server model are the same.

```c
esp_ble_mesh_server_model_send_msg(model, ctx, ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS,
                                   msg->len ,msg->data );
```

`esp_ble_mesh_model_publish` The destination address of the published message is the publish address of the model binding.
model need to binding publish address,Then the node that subscribes to this address will receive the message.
```c
err = esp_ble_mesh_model_publish(model, ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS,
                                msg->len ,msg->data ROLE_NODE);
```

