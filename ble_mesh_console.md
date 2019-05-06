# Introduction

This demo helps you to quickly implement the ESP BLE Mesh functionality in your applications. For example, you can implement this demo on your device working as a BLE Mesh Provisioner to provisioning unprovisioned devices, so they can join a mesh network, or working as a node to control the state of lights.

Before starting, you could enter the `help` command in your serial port tool to check all the commands that are already supported in this demo.


# Project Structure

The folder `ble_mesh_provisioner` contains the following files and subfolders:

| File Name        |Description               |
| ----------------------|------------------------- |
| `ble_mesh_adapter`      | Adapts the codes for initialization by using the console. |
| `ble_mesh_cfg_srv_model` | Stores all the structs related to the initialization of models. |
| `ble_mesh_console_lib` | Stores utilities for converting data formats. |
| `ble_mesh_console_main` | The main function that initializes the console and registers all the commands |
| `ble_mesh_console_system`  | Implements the system-related commands: `restart`, `free`, `make`. |
| `ble_mesh_reg_cfg_client_cmd`| Implements the **Configuration Client** model related command: `bmccm`. |
| `ble_mesh_reg_gen_onoff_client_cmd`| Implements the **Generic OnOff Client** model related command: `bmgocm`.|
| `ble_mesh_reg_test_perf_client_cmd` | Implements the performance test related command: `bmcperf`. |
| `ble_mesh_register_node_cmd`  | Implements the node related commands: `bmreg`, `bmoob`, `bminit`, `bmpbearer`, `bmtxpower`.  |
| `ble_mesh_register_provisioner_cmd` | Implements the provisioner related commands: `bmpreg`, `bmpdev`, `bmpbearer`, `bmpgetn`, `bmpaddn`,`bmpbind`,`bmpkey`. |
| `register_bluetooth`    | Implements the command to get the MAC address of a BLE device: `btmac` |

# What You Need

Download and flash the `ble_mesh_console/ble_mesh_provisioner` project to your ESP32 development board and then use commands below to implement this demo.

## Implement the demo as a provisioner

The commands that are related to implementing this demo as a provisioner are shown in the table below:


| Command        |Description               |
| ----------------------|------------------------- |
| `bmreg`  | ble mesh: provisioner/node register callback |
| `bmpreg` | ble mesh provisioner: register callback |
| `bmoob`  | ble mesh: provisioner/node config OOB parameters|
| `bminit` | ble mesh: provisioner/node init |
| `bmpbearer`  | ble mesh provisioner: enable/disable different bearers |
| `bmpdev`| ble mesh provisioner: add/delete unprovisioned device |

Please follow the steps below to implement this demo as a provisioner. 

1. Enter `bmreg` to register the `ble_mesh_prov_cb` and `ble_mesh_model_cb` callback functions.
    * `ble_mesh_prov_cb` handles all events triggered when your board is provisioning other devices. 
    * `ble_mesh_model_cb` handles all events triggered when any server model of your board receives or sends any message. 

2. Enter `bmpreg` to initialize your board as a provisioner, which registers the `ble_mesh_prov_adv_cb` callback function.
    * `ble_mesh_prov_adv_cb` will be triggered when your board receives an unprovisioned Device beacon broadcasted by an unprovisioned device.

3. Enter `bmoob -p 0x5` to initialize all the information needed for a device to be a provisioner. 
    * The parameter `-p` stands for provisioner and its value `0x5` indicates the starting address assigned to the node by the provisioner.

4. Enter `bminit -m 0x01` to initialize the model that is necessary for the provisioner's provisioning. 
    * The parameter `-m` stands for model and its value `0x01` indicates the model ID of the **Configuration Client** model. 
    
    After this command, the initialization now has been completed.

5. Enter `bmpbearer -b 0x3 -e 1` to enable bearers, which include the PB-ADV bearer and the PB-GATT bearer. 
    * The parameter `-b` stands for "bearer" and its value `0x3` indicates both the PB-ADV bearers and the PB-GATT bearers are enabled; 
    * The parameter `-e` stands for "enable" and its value `1` indicates to enable the bearer. 
    
    After this command, the provisioner now can send and receive messages.

6. Enter `bmpdev -z add -d MAC_address -b 0x3 -a 0 -f 1` to start provisioning an unprovisioned device.
    
    * The parameter `MAC_address` indicates the MAC address of the unprovisioned device. Here, you can use the `btmac` command to query the unprovisioned device's MAC address. 
    * `-f 1` indicates that the device will be immediately provisioning.


## 2.2 Implement the demo as a node

The commands that are related to implementing this demo as a node are shown in the table below:


| Command    |Description               |
| -------------|------------------------- |
| `bmreg`  | Adapt the original initialization method to the way it is applied to the console. |
| `bmoob`  | ble mesh: provisioner/node config OOB parameters |
| `bminit` | ble mesh: provisioner/node init |
| `bmnbearer` | ble mesh node: enable/disable different bearers |

Please follow the steps below to implement this demo as a node.

1. Enter `bmreg` to register the `ble_mesh_prov_cb` and `ble_mesh_model_cb` callback functions.
    * `ble_mesh_prov_cb` handles all events triggered when your board is provisioning other devices. 
    * `ble_mesh_model_cb` handles all events triggered when any server model of your board receives or sends any message. 
    
3. Enter `bmoob -o 0 -x 0` to initialize the OOB information of your board as a node

3. Enter `bminit -m 0x1000` to initialize the model that is necessary for the provisioner's provisioning. 
    * The parameter `-m` stands for model and its value `0x1000` indicates the model ID of the **Generic Onoff Server** model. 

4. Enter `bmpbearer -b 0x3 -e 1` to enable bearers, which include the PB-ADV bearer and the PB-GATT bearer. 
    * The parameter `-b` stands for "bearer" and its value `0x3` indicates both the PB-ADV bearers and the PB-GATT bearers are enabled; 
    * The parameter `-e` stands for "enable" and its value `1` indicates to enable the bearer. 
    
    After this command, the provisioner now can send and receive messages.



# Example Walkthrough

## Main Entry Point

The programâ€™s entry point is the `app_main()` function.

## Initialization

The code block below first initializes the device's Bluetooth-related functions (including the Bluetooth and BLE Mesh), and its console.

### Initializing the Bluetooth

```c
esp_err_t res;
    nvs_flash_init();
    // init and enable bluetooth
    res = bluetooth_init();
    if (res) {
        printf("esp32_bluetooth_init failed (ret %d)", res);
```        

This demo calls the `bluetooth_init` function to:

1. First initialize the non-volatile storage library, which allows saving key-value pairs in flash memory and is used by some components. You can save the node's keys and configuration information at `menuconfig` --> `Bluetooth Mesh support` --> `Store Bluetooth Mesh key and configuration persistently`:

     ```c
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK( ret );
     ```

2. Initializes the BT controller by first creating a BT controller configuration structure named `esp_bt_controller_config_t` with default settings generated by the `BT_CONTROLLER_INIT_CONFIG_DEFAULT()` macro. The BT controller implements the Host Controller Interface (HCI) on the controller side, the Link Layer (LL) and the Physical Layer (PHY). The BT Controller is invisible to the user applications and deals with the lower layers of the BLE stack. The controller configuration includes setting the BT controller stack size, priority and HCI baud rate. With the settings created, the BT controller is initialized and enabled with the `esp_bt_controller_init()` function:

    ```c
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ret = esp_bt_controller_init(&bt_cfg);
    ```

    Next, the controller is enabled in BLE Mode. 

    ```c
    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    ```
    The controller should be enabled in `ESP_BT_MODE_BTDM`, if you want to use the dual mode (BLE + BT). There are four Bluetooth modes supported:
    
    * `ESP_BT_MODE_IDLE`: Bluetooth not running
    * `ESP_BT_MODE_BLE`: BLE mode
    * `ESP_BT_MODE_CLASSIC_BT`: BT Classic mode
    * `ESP_BT_MODE_BTDM`: Dual mode (BLE + BT Classic)

    After the initialization of the BT controller, the Bluedroid stack, which includes the common definitions and APIs for both BT Classic and BLE, is initialized and enabled by using:

    ```c
    ret = esp_bluedroid_init();
    ret = esp_bluedroid_enable();
    ```

### Initializing the Console

This demo calls the `initialize_console()` function: 

```c
#if CONFIG_STORE_HISTORY
    initialize_filesystem();
#endif
    initialize_console();
```

### Registering Commands

This demo runs the following codes to register commands:
 
```c
    /* Register commands */
    esp_console_register_help_command();
    register_system();
    register_bluetooth();
    ble_mesh_register_mesh_node();
    ble_mesh_register_mesh_test_performance_client();
#if (CONFIG_BT_MESH_GENERIC_ONOFF_CLI)
    ble_mesh_register_gen_onoff_client();
#endif
#if (CONFIG_BT_MESH_PROVISIONER)
    ble_mesh_register_mesh_provisioner();
#endif
#if (CONFIG_BT_MESH_CFG_CLI)
    ble_mesh_register_configuration_client_model();
#endif
```

##  Main Program

The main program constantly reads data from the command line. `esp_console_run` will parse the argument and call the callback function of the previously initialized command.

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
        ...
        /* linenoise allocates line buffer on the heap, so need to free it */
        linenoiseFree(line);
    }
    return;
}
```
