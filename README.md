# 1. Introduction
## 1.1 Demo Function
1. This demo forward the packet sent by the app.
2. The destination address of the forwarded packet is entered by the user.
3. Forwarded message types include `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET`,`ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET`,`ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK`.
4. Report node's onoff state message `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS`.

example: App send `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` messge to the node（ble_mesh_client_model）.Then node will send `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` message to other node（ble_mesh_node） that the destination address is the address entered by the serial port.

## 1.1.1 requirement
1. One device run ble_mesh_client_model project.
2. One device run ble_mesh_node project.
3. You can use nRF Mesh app to control two device

## 1.2 Node Composition
This demo has only one element, in which the following three models are implemented:
- **Configuration Server model**: The role of this model is mainly to configure Provisioner device’s AppKey and set up its relay function, TTL size, subscription, etc.
- **Generic OnOff Client model**: This model implements the most basic function of turning the lights on and off.
- **Generic OnOff Server model**: This model implements the node's onoff state.

## 1.3 Message Interaction

You can choose the following 4 ways to interact：
1. Acknowledged Get
2. Acknowledged Set
3. Unacknowledged Set
4. Period publishing

![Packet interaction](images/2.gif)

## 2. Code Analysis

### 2.1  model definition

#### 2.1.1 Generic OnOff Server model definition
```c
//model publish init,Allocating space to publish message.
static esp_ble_mesh_model_pub_t onoff_srv_pub = {
    .msg = NET_BUF_SIMPLE(2 + 1),
    .update = NULL,        
    .dev_role = MSG_ROLE,
};
//registe massage opcode
static esp_ble_mesh_model_op_t onoff_op[] = {
    { ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET, 0, 0},
    { ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET, 2, 0},
    { ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK, 2, 0},
    ESP_BLE_MESH_MODEL_OP_END,
};
//registe onoff server model.
static esp_ble_mesh_model_t root_models[] = {
    //onoff server's onoff state init
    ESP_BLE_MESH_SIG_MODEL(ESP_BLE_MESH_MODEL_ID_GEN_ONOFF_SRV, onoff_op,
    &onoff_srv_pub, &led_state[0]),
};
```
#### 2.1.2 Generic OnOff Client model definition
```c
//model publish init,Allocating space to publish message.
static esp_ble_mesh_model_pub_t onoff_cli_pub = {
    .msg = NET_BUF_SIMPLE(2 + 1),
    .update = NULL,
    .dev_role = MSG_ROLE,
};
//registe onoff client model.
static esp_ble_mesh_model_t root_models[] = {
    ESP_BLE_MESH_MODEL_GEN_ONOFF_CLI(&onoff_cli_pub, &onoff_client),
};
```
### 2.2 model callback
#### 2.2.1 onoff client model callback
```c
esp_ble_mesh_register_generic_client_callback(esp_ble_mesh_generic_cb);

```
1. Callback trigger.
>Receiving a message about onoff client mode will trigger this callback function
>call send messgae API about onoff client model.

2. The event that the callback function needs to handle.

| Event name    | Opcode      |Description                                 |
| ------------- | ------------|------------------------------------------- |
| ESP_BLE_MESH_GENERIC_CLIENT_GET_STATE_EVT   | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET    |onoff client model receive messages from onoff server model, receive ackownleged before send `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET` message        |
| ESP_BLE_MESH_GENERIC_CLIENT_SET_STATE_EVT   | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET    | onoff client model receive messages from onoff server model , receive ackownleged before send `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` message  |
| ESP_BLE_MESH_GENERIC_CLIENT_PUBLISH_EVT     | no deal opcode                         | receive publish message    |
| ESP_BLE_MESH_GENERIC_CLIENT_TIMEOUT_EVT     | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET    | send message timeout event,send  `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` message timeout event   |
  

#### 2.2.2 onoff server callback
```c
esp_ble_mesh_register_custom_model_callback(esp_ble_mesh_model_cb);

```
1. Callback trigger.
>Receiving a message about onoff server mode will trigger this callback function
>call send messgae API about onoff server model.
2. The event that the callback function needs to handle.


| Event name    | Opcode      |Description                                 |
| ------------- | ------------|------------------------------------------- |
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET        |onoff server model receive messages from onoff client model.opcode registered before （static `esp_ble_mesh_model_op_t` onoff_op[]）     |
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET        | onoff server model receive messages from onoff client model.opcode registered before （static `esp_ble_mesh_model_op_t` onoff_op[]）  |
| ESP_BLE_MESH_MODEL_OPERATION_EVT   | ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK  |  onoff server model receive messages from onoff client model.opcode registered before （static `esp_ble_mesh_model_op_t` onoff_op[]）   |
| ESP_BLE_MESH_MODEL_SEND_COMP_EVT   | no deal opcode                           | send message timeout event | `ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET` message timeout event   |Call `esp_ble_mesh_server_model_send_msg` API will trigger this event when it call completion|
| ESP_BLE_MESH_MODEL_PUBLISH_COMP_EVT| no deal opcode       | Call `esp_ble_mesh_model_publish` API will trigger this event when it call completion     |
  

### 2.3 model send messgae
#### 2.3.0 message contorl
`esp_ble_mesh_set_msg_common` This function used to set message contorl parameters. 
| parameter name        |Description               |
| ----------------------|------------------------- |
| `opcode`      | The Message opcode  |
| `model`       | Pointer to the client model structure  |
| `ctx.net_idx` | NetKey Index of the subnet through which to send the message. |
| `ctx.app_idx` | AppKey Index for message encryption. |
| `ctx.addr`    | Remote address/Destination address |
| `ctx.send_ttl`| message TTL,relay parameter |
| `ctx.send_rel`| Force sending reliably,Waiting for a response from the message   |
| `msg_timeout` | Time to wait for a response   |
| `msg_role`    | message role (node/proviser)  |
**note:After the message is sent，you should check the event (ESP_BLE_MESH_MODEL_SEND_COMP_EVT),Check if the message was sent successfully**

#### 2.3.1 onoff client send messgae
`esp_ble_mesh_generic_client_get_state` This API used to client model get the state from the server model.such as onoff state.
```c
esp_ble_mesh_generic_client_get_state_t get_state = {0};
esp_ble_mesh_set_msg_common(&common, node, onoff_client.model, ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET);
err = esp_ble_mesh_generic_client_get_state(&common, &get_state);
if (err) {
    ESP_LOGE(TAG, "%s: Generic OnOff Get failed", __func__);
    return;
}
```

`esp_ble_mesh_generic_client_set_state`This API used to client model set the state from the server model.such as onoff state.
```c
esp_ble_mesh_generic_client_set_state_t set_state = {0};
esp_ble_mesh_set_msg_common(&common, &set_state, onoff_client.model,
                             ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET, remote_onoff);
err = esp_ble_mesh_generic_client_set_state(&common, &set_state);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "%s: Generic OnOff Set failed", __func__);
    return;
}
```

#### 2.3.2 onoff server send messgae

`esp_ble_mesh_server_model_send_msg` Model needs to bind appkey before sending a message.
```c
err = esp_ble_mesh_server_model_send_msg(model, ctx, ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS,
       sizeof(send_data), &send_data);
```
`esp_ble_mesh_model_publish` The destination address of the published message is the publish address of the model binding.
model need to binding publish address,Then the node that subscribes to this address will receive the message.

```c
err = esp_ble_mesh_model_publish(model, ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS,
                                 sizeof(led->current), &led->current, ROLE_NODE);
```


## 2.4 receive command by uart

You should use the serial port tool.Connect the pins of the device 16,17.

```c
#define UART1_TX_PIN  GPIO_NUM_16
#define UART1_RX_PIN  GPIO_NUM_17
```
There is a Task here that receive command by uart.
You can enter the address of another node as the destination address for the message.
`remote_addr` that represents the destination address of the packet you are forwarding.
such as:input 5,then The value of this variable is 0x05.

```c
static void board_uart_task(void *p)
{   
    uint8_t *data = calloc(1, UART_BUF_SIZE);
    uint32_t input;
    
    while (1) { 
        int len = uart_read_bytes(MESH_UART_NUM, data, UART_BUF_SIZE, 100 / portTICK_RATE_MS);
        if (len > 0) {
            input = strtoul((const char *)data, NULL, 16);
            remote_addr = input & 0xFFFF;
            ESP_LOGI(TAG, "%s: input 0x%08x, remote_addr 0x%04x", __func__, input, remote_addr);
            memset(data, 0, UART_BUF_SIZE);
        }
    }
    
    vTaskDelete(NULL);
}
```


## 2.5 timing diagram
The timing diagram is shown below：

![Packet interaction](images/picture5.png) <div align=center></div>

> * App provising unprovisioned devices to node.
> * App add appkey to the node and bind appkey with generic onoff server an generic onoff client model.
> * App send control message,then node forward the message to other node. 

**note：The node does not send a message immediately after entering the address through the serial port.
When nRF_Mesh_App sends a control message to the node, the client node sends a message to the node of the previously entered message.**



## 2.6 Use nRF_Mesh_App

![Packet interaction](images/app.png)

> * As shown in the note 1 above,Scan unprovisioned devices.
> * As shown in the note 3 above,provising unprovisioned devices.
> * As shown in the note 5 above,click CONFOG button,Then you can config node's model.
> * As shown in the note 6 above,click Generic On Off Client button.
> * As shown in the note 7 above,bind appkey to Generic On Off Client model.
> * As shown in the note 9 above,bind appkey to Generic On Off Server model.
> * As shown in the note 10 above,control Generic On Off Server model's state.

