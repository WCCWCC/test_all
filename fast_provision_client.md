# 1. Introduction
## 1.1 Demo Function
Provisioning an unprovisioned devices to become node.

Bind appkey for the provisioner's own mode.

Obtain the addresses of all nodes in the fast provision network and control the nodes through the group address.

**Note:The demo's functionality is consistent with the EspBleMesh app.**

## 1.2 Node Composition
This demo has only one element, in which the following four models are implemented:
- **Configuration Server model**： This model is used to represent a mesh network configuration of a device.
- **Configuration Client model**: This model is used to represent an element that can control and monitor the configuration of a node.
- **Generic OnOff Client model** controls a Generic OnOff Server via messages defined by the Generic OnOff Model, that is, turning on and off the lights.

- **Vender Client model**:This model is used to control the `fast_prov_server` state.

**Note:The use of model can refer to the documentation of other demos.**

## 2. Code Analysis
### 2.1  Data Structure
`example_prov_info_t` is used to control the keys, address range, and number of network nodes of the mesh network.
```c
typedef struct {
    uint16_t net_idx;       /*  Netkey index value */
    uint16_t app_idx;       /*  AppKey index value */
    uint8_t  app_key[16];   /*  Use the same app_key throughout the network*/

    uint16_t node_addr_cnt; /* Number of BLE Mesh nodes in the network，
                               Consistent with the number of devices entered by the app */
    uint16_t unicast_min;   /* Minimum unicast address to be assigned to the nodes in the network */
    uint16_t unicast_max;   /* Maximum unicast address to be assigned to the nodes in the network */
    uint16_t group_addr;    /* This group address is used to control the switch of all node lights 
                                                                                at the same time. */
    uint8_t  match_val[16]; /* Match value used by Fast Provisoning Provisioner */
    uint8_t  match_len;

    uint8_t  max_node_num;  /* Maximum number of nodes can be provisioned by the client */
} __attribute__((packed)) example_prov_info_t;
```
### 2.2  Code Flow
I will follow the order in which the code runs，Introduce related events and APIs

### 2.2.1 ble_mesh_init 

Filter received uuid values by setting match values. Uuid has 16 bytes and sets the matching value of the corresponding byte. If the uuid matches, the `ESP_BLE_MESH_PROVISIONER_RECV_UNPROV_ADV_PKT_EVT` event will be triggered.If the uuid not matches,The uuid of the unprovisioned device will be ignored.
```c
err = esp_ble_mesh_provisioner_set_dev_uuid_match(match, 0x02, 0x00, false);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to set matching device UUID", __func__);
    return ESP_FAIL;
}
```
After the provisioner is initialized, there is no appkey. You need to add it through the api(`esp_ble_mesh_provisioner_add_local_app_key`) to make sure that the addition is successful.
```c
err = esp_ble_mesh_provisioner_add_local_app_key(prov_info.app_key, prov_info.net_idx, prov_info.app_idx);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to add local application key", __func__);
    return ESP_FAIL;
}
```
Bind appkey for the provisioner's own model.If you want to send a message between the server model and the client model of different nodes, you must first bind the appkey to the model.
In this demo，appkey is bound to onoff client model and vender client model.
The API of `esp_ble_mesh_provisioner_add_local_app_key` will trigger the `ESP_BLE_MESH_PROVISIONER_ADD_LOCAL_APP_KEY_COMP_EVT` event

```c
prov_info.app_idx = param->provisioner_add_app_key_comp.app_idx;
err = esp_ble_mesh_provisioner_bind_app_key_to_local_model(PROV_OWN_ADDR, prov_info.app_idx,
                                              ESP_BLE_MESH_MODEL_ID_GEN_ONOFF_CLI, CID_NVAL);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to bind AppKey with OnOff Client Model", __func__);
    return;
}
err = esp_ble_mesh_provisioner_bind_app_key_to_local_model(PROV_OWN_ADDR, prov_info.app_idx,
                                            ESP_BLE_MESH_VND_MODEL_ID_FAST_PROV_CLI, CID_ESP);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to bind AppKey with Fast Prov Client Model", __func__);
    return;
}
```
### 2.2.2 provisioning device
Unprovisioned device will continuously send Unprovisioned Device beacon packets.The packet contains the value of uuid.If the uuid matches, the `ESP_BLE_MESH_PROVISIONER_RECV_UNPROV_ADV_PKT_EVT` event will be triggered.
In this event,provisioner will add unprovisioned device info to the unprov_dev queue.
```c
err = esp_ble_mesh_provisioner_add_unprov_dev(&add_dev, flag);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to start provisioning a device", __func__);
    return;
}

if (!reprov) {
    if (prov_info.max_node_num) {
         prov_info.max_node_num--;
}
}
```
### 2.2.3 send cache data
If provisioner provisioning done,This event will be triggered `ESP_BLE_MESH_PROVISIONER_PROV_COMPLETE_EVT`
In this event,Add appkey to node's config server model.Appkey is one of the data that needs to be cached as a provisioner.
If a device wants to have the ability to provisioning other unprovisioned device，It must have the corresponding cached data.

The API of esp_ble_mesh_config_client_set_state will trigger `ESP_BLE_MESH_CFG_CLIENT_SET_STATE_EVT` event after successful call，`ESP_BLE_MESH_CFG_CLIENT_TIMEOUT_EVT` is triggered when the call times out.

```c
common.opcode       = ESP_BLE_MESH_MODEL_OP_APP_KEY_ADD;
common.model        = model;
common.ctx.net_idx  = info->net_idx;
common.ctx.app_idx  = 0x0000; /* not used for config messages */
common.ctx.addr     = info->dst;
common.ctx.send_rel = false;
common.ctx.send_ttl = 0;
common.msg_timeout  = info->timeout;
common.msg_role     = info->role;

return esp_ble_mesh_config_client_set_state(&common, &set);
```

In this event (`ESP_BLE_MESH_CFG_CLIENT_SET_STATE_EVT`),The provisioner continues to send the cache information needed by the device to have the provisioning capabilities.The cached data type is defined in the structure (`example_fast_prov_info_set_t`).
Now,The cached data required as a provisioner is sent.
when the api (`example_send_fast_prov_info_set`) call times out,`ESP_BLE_MESH_CLIENT_MODEL_SEND_TIMEOUT_EVT` event will trigger.

```c
err = example_send_fast_prov_info_set(fast_prov_client.model, &info, &set);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to set Fast Prov Info Set message", __func__);
    return;
}
```
Calling this api(`example_send_fast_prov_info_set`) will send a message with opcode `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_SET`.
The response to this message will trigger `ESP_BLE_MESH_MODEL_OPERATION_EVT` event with opcode `ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS` .

### 2.2.4 control node's light
In this event (`ESP_BLE_MESH_MODEL_OPERATION_EVT`) and will receive message with opcode (`ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_INFO_STATUS`).
The provisioner will start a timer，Used to get the address of all nodes in the mesh network.
```c
        ESP_LOG_BUFFER_HEX("fast prov info status", data, len);
#if !defined(CONFIG_BT_MESH_FAST_PROV)
        prim_prov_addr = ctx->addr;
        k_delayed_work_init(&get_all_node_addr_timer, example_get_all_node_addr);
        k_delayed_work_submit(&get_all_node_addr_timer, GET_ALL_NODE_ADDR_TIMEOUT);
#endif
        break;
```
Behavior after the timer expires,The provisioner will send a message with opcode. (`ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_GET`)
```c
err = example_send_fast_prov_all_node_addr_get(model, &info);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to send Fast Prov Node Address Get message", __func__);
    return;
}
```

The provisioner will receive a acknowledge. A message with opcode(`ESP_BLE_MESH_VND_MODEL_OP_FAST_PROV_NODE_ADDR_STATUS`).
The provisioner will turn on the lights of all nodes by group address.

```c
example_msg_common_info_t info = {
    .net_idx = node->net_idx,
    .app_idx = node->app_idx,
    .dst = node->group_addr,
    .timeout = 0,
    .role = ROLE_PROVISIONER,
};
err = example_send_generic_onoff_set(cli_model, &info, LED_ON, 0x00, false);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Failed to send Generic OnOff Set Unack message", __func__);
    return ESP_FAIL;
}
```
