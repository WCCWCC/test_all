# 1. Introduction
## 1.1 Demo Function

This is a demo that wifi and bluetooth coexist.

Testing the transfer rate of wifi when Bluetooth is working properly.

The function of Bluetooth is the function of `ble_mesh_fast_prov_server`.

# 2. How to use this demo
You need to download the project(`ble_mesh_wifi_coexist`) code to board.
Enter the following command by connecting to the borad terminal：
1. run `sta ssid password` in board terminal.
If you connect to the wifi named `tset_wifi` and the wifi password `12345678`.You should enter the command `sta tset_wifi 12345678`

2. run `iperf -s -i 3` in board terminal.
This command starts a tcp server to test the transfer rate of wifi.

3. run `iperf -c 192.168.10.42 -i 3 -t 60` in PC terminal.

4. You can use Bluetooth at the same time.Control node light switch.

# 3. Project Structure
The folder `ble_mesh_wifi_coexist` contains the following files and subfolders:

```
$ tree examples/bluetooth/ble_mesh/ble_mesh/ble_mesh_wifi_coexist
├── main        /* Stores the `.c` and `.h` application code files for this demo */
├── components  /* Stores the `.c` and `.h` iperf code files for this demo */
├── Makefile    /* Compiling parameters for the demo */
├── README.md   /* Quick start guide */
├── build
├── sdkconfig      /* Current parameters of `make menuconfig` */
├── sdkconfig.defaults   /* Default parameters of `make menuconfig` */
├── sdkconfig.old      /* Previously saved parameters of `make menuconfig` */
└── tutorial         /* More in-depth information about the demo */
```
This folder `main` mainly implement the code of the Bluetooth application layer.The Bluetooth function is the same as `ble_mesh_fast_prov_server`.

This folder `components` Mainly implement the code of the wifi application layer.
The wifi part implements some basic commands and test commands corresponding to `iperf`.As the following command:


Note:`iperf` is a network performance testing tool. Iperf can test maximum TCP and UDP bandwidth performance with multiple parameters and UDP features that can be adjusted as needed to report bandwidth, delay jitter, and packet loss.

# Example Walkthrough
## Main Entry Point
The program’s entry point is the app_main() function:

Initialize bluetooth related functions, then initialize the wifi console.

```c
void app_main(void)
{
    esp_err_t err;

    ESP_LOGI(TAG, "Initializing...");

    err = board_init();
    if (err) {
        ESP_LOGE(TAG, "board_init failed (err %d)", err);
        return;
    }

    err = bluetooth_init();
    if (err) {
        ESP_LOGE(TAG, "esp32_bluetooth_init failed (err %d)", err);
        return;
    }

    /* Initialize the Bluetooth Mesh Subsystem */
    err = ble_mesh_init();
    if (err) {
        ESP_LOGE(TAG, "Bluetooth mesh init failed (err %d)", err);
        return;
    }

    wifi_console_init();
}
```
## board_init
`board_init` starts by initializing the gpio pin of the led light.
Initialize the `led_state[i].previous` value to `LED_OFF`.
```c
for (int i = 0; i < 3; i++) {
    gpio_pad_select_gpio(led_state[i].pin);
    gpio_set_direction(led_state[i].pin, GPIO_MODE_OUTPUT);
    gpio_set_level(led_state[i].pin, LED_OFF);
    led_state[i].previous = LED_OFF;
}
```
`bluetooth_init` also created a task named `led_action_thread` to control the status of the light.
`bluetooth_init` also created a queue named `led_action_queue` is used to store data.The format of the data is `struct _led_state`.
```c
led_action_queue = xQueueCreate(60, sizeof(struct _led_state));
ret = xTaskCreate(led_action_thread, "led_action_thread", 4096, NULL, 5, NULL);
```
The task continuously gets data from the queue(`led_action_queue`).Set the state of the light by the value obtained from the queue
Set the state of the light by calling the following function `gpio_set_level`.

```c
static void led_action_thread(void *arg)
{
    struct _led_state led = {0};

    while (1) {
        if (xQueueReceive(led_action_queue, &led, (portTickType)portMAX_DELAY)) {
            ESP_LOGI(TAG, "%s: pin 0x%04x onoff 0x%02x", __func__, led.pin, led.current);
            /* If the node is controlled by phone, add a delay when turn on/off led */
            if (fast_prov_server.primary_role == true) {
                vTaskDelay(50 / portTICK_PERIOD_MS);
            }
            gpio_set_level(led.pin, led.current);
        }
    }
}
```

## bluetooth_init
`bluetooth_init` starts by initializing the non-volatile storage library. This library allows to save key-value pairs in flash memory and is used by some components such as the Wi-Fi library to save the SSID and password.You can use the option to configure menuconfig to save the node's key and configuration information. `Bluetooth Mesh support`  ---> `Store Bluetooth Mesh key and configuration persistently`:

```c
esp_err_t ret = nvs_flash_init();
if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
}
ESP_ERROR_CHECK( ret );
```

`bluetooth_init` also initializes the BT controller by first creating a BT controller configuration structure named `esp_bt_controller_config_t` with default settings generated by the `BT_CONTROLLER_INIT_CONFIG_DEFAULT()` macro. The BT controller implements the Host Controller Interface (HCI) on the controller side, the Link Layer (LL) and the Physical Layer (PHY). The BT Controller is invisible to the user applications and deals with the lower layers of the BLE stack. The controller configuration includes setting the BT controller stack size, priority and HCI baud rate. With the settings created, the BT controller is initialized and enabled with the `esp_bt_controller_init()` function:

```c
esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
ret = esp_bt_controller_init(&bt_cfg);
```

Next, the controller is enabled in BLE Mode. 

```c
ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
```
>The controller should be enabled in `ESP_BT_MODE_BTDM`, if you want to use the dual mode (BLE + BT).
 
There are four Bluetooth modes supported:

1. `ESP_BT_MODE_IDLE`: Bluetooth not running
2. `ESP_BT_MODE_BLE`: BLE mode
3. `ESP_BT_MODE_CLASSIC_BT`: BT Classic mode
4. `ESP_BT_MODE_BTDM`: Dual mode (BLE + BT Classic)

After the initialization of the BT controller, the Bluedroid stack, which includes the common definitions and APIs for both BT Classic and BLE, is initialized and enabled by using:

```c
ret = esp_bluedroid_init();
ret = esp_bluedroid_enable();
```

## ble_mesh_init
```c
static esp_err_t ble_mesh_init(void)
{
    esp_err_t err;

    /* First two bytes of device uuid is compared with match value by Provisioner */
    memcpy(dev_uuid + 2, esp_bt_dev_get_address(), 6);

    esp_ble_mesh_register_prov_callback(example_ble_mesh_provisioning_cb);
    esp_ble_mesh_register_custom_model_callback(example_ble_mesh_custom_model_cb);
    esp_ble_mesh_register_config_client_callback(example_ble_mesh_config_client_cb);
    esp_ble_mesh_register_config_server_callback(example_ble_mesh_config_server_cb);

    err = esp_ble_mesh_init(&prov, &comp);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "%s: Failed to initialize BLE Mesh", __func__);
        return err;
    }
    ...
    return ESP_OK;
}
```
`ble_mesh_init` starts by initializing the device's uuid (`dev_uuid`).Uuid is used to distinguish devices when provisioning.

- `esp_ble_mesh_register_prov_callback(esp_ble_mesh_prov_cb)`: registers the provisioning callback function in the BLE Mesh stack.`esp_ble_mesh_prov_cb` is used to handle events thrown by protocol stations.This requires the user to implement it himself, and also needs to know the meaning of the event and how to trigger it.
For example: The event `ESP_BLE_MESH_PROVISIONER_PROV_LINK_OPEN_EVT`will trigger when provisioner start provisioning unprovisioned device. If you want to handle this event,You need to implement the handler for this event `example_ble_mesh_provisioning_cb`.You also need to register this event handler with the protocol station by calling the API `esp_ble_mesh_register_prov_callback`.
```c
ESP_BLE_MESH_PROVISIONER_PROV_LINK_OPEN_EVT,                /*!< Provisioner establish a BLE Mesh link event */

static void example_ble_mesh_provisioning_cb(esp_ble_mesh_prov_cb_event_t event,
        esp_ble_mesh_prov_cb_param_t *param)
{
    esp_err_t err;
    
    switch (event) {
    case  :
        ESP_LOGI(TAG, "ESP_BLE_MESH_PROVISIONER_PROV_LINK_OPEN_EVT");
        provisioner_prov_link_open(param->provisioner_prov_link_open.bearer);
        break;
    }
}
esp_ble_mesh_register_prov_callback(example_ble_mesh_provisioning_cb);
```
- `esp_ble_mesh_register_custom_model_callback(esp_ble_mesh_model_cb)`: Register BLE Mesh callback for user-defined models' operations. 
- `esp_ble_mesh_register_config_client_callback(example_ble_mesh_config_client_cb)`:Register BLE Mesh Config Client Model callback.
- `esp_ble_mesh_register_config_server_callback(example_ble_mesh_config_server_cb)`:Register BLE Mesh Config Server Model callback.

- `esp_ble_mesh_init(&prov, &comp)` ：Initialize BLE Mesh module.This API initializes provisioning capabilities and composition data information.Registered information is stored in the structure `prov`.The structure `prov` is essentially a composition of one or more models.

## wifi_console_init
`wifi_console_init` starts by initializing 
```c
    initialise_wifi();
    initialize_console();

    /* Register commands */
    esp_console_register_help_command();
    register_wifi();
```

```c
    /* Main loop */
    while (true) {
        /* Get a line using linenoise.
         * The line is returned when ENTER is pressed.
         */
        char *line = linenoise(prompt);
        if (line == NULL) { /* Ignore empty lines */
            continue;
        }
        /* Add the command to the history */
        linenoiseHistoryAdd(line);

        /* Try to run the command */
        int ret;
        esp_err_t err = esp_console_run(line, &ret);
        if (err == ESP_ERR_NOT_FOUND) {
            printf("Unrecognized command\n");
        } else if (err == ESP_OK && ret != ESP_OK) {
            printf("Command returned non-zero error code: 0x%x\n", ret);
        } else if (err != ESP_OK) {
            printf("Internal error: %s\n", esp_err_to_name(err));
        }
        /* linenoise allocates line buffer on the heap, so need to free it */
        linenoiseFree(line);
    }
    return;
}
```


