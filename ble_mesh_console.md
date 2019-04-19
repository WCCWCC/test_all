# 1. Introduction
## 1.1 Demo Function

1. This is a demo that help you quickly implement BLE MESH related functions
2. You can implement the provisioner to control the device to access the mesh network.
3. You can implement node to control the state of the light.

Noet:In this demo,You can use the `help` command to see all the commands currently supported.You can also modify the code and add the commands you want.

# 2. How to use this demo
You need to download the project(`ble_mesh_console/ble_mesh_provisioner`) code to board.

## 2.1 Implement a provisioner
The implementationer related commands are shown in the table below：

| File Name        |Description               |
| ----------------------|------------------------- |
| `bmreg`  | ble mesh: provisioner/node register callback |
| `bmpreg` | ble mesh provisioner: register callback |
| `bmoob`  | ble mesh: provisioner/node config OOB parameters|
| `bminit` | ble mesh: provisioner/node init |
| `bmpbearer `  | ble mesh provisioner: enable/disable different bearers |
| `bmpdev `| ble mesh provisioner: add/delete unprovisioned device |

For example:
1. bmreg        
This command should be executed before other commands are run.This command registers the callback function `ble_mesh_prov_cb` and `ble_mesh_model_cb`.`ble_mesh_prov_cb`. When the device is provisioning, it triggers a series of events that will be processed in the `ble_mesh_prov_cb` function.The callback function `ble_mesh_model_cb` is triggered when the server model in the node receives or sends a message.
2. bmpreg

3. bmoob -p 0x5
4. bminit -m 0x01
5. bmpbearer -b 0x3 -e 1
6. bmpdev -z add -d `MAC` -b 0x3 -a 0 -f 1      

** Note: `MAC` Is the address of the Bluetooth device，You can use `btmac` to query the device address on devices that need to access the network. **

## 2.2 Implement a node

| File Name    |Description               |
| -------------|------------------------- |
| `bmreg`  | Adapt the original initialization method to the way it is applied to the console. |
| `bmoob`  | ble mesh: provisioner/node config OOB parameters |
| `bminit` | ble mesh: provisioner/node init |
| `bmnbearer` | ble mesh node: enable/disable different bearers |

For example:
1. bmreg
2. bmoob -o 0 -x 0
3. bminit -m 0x1000
4. bmnbearer -b 0x3 -e 1

# 3. Project Structure
The folder `ble_mesh_provisioner` contains the following files and subfolders:

| File Name        |Description               |
| ----------------------|------------------------- |
| `ble_mesh_adapter`      | Adapt the original initialization method to the way it is applied to the console. |
| `ble_mesh_cfg_srv_model` | Implemented the definition and initialization of the model related structure. |
| `ble_mesh_console_lib` | Implemented common format conversion functions. |
| `ble_mesh_console_main` | The main function, the initialization of the console and the registration of the command |
| `ble_mesh_console_system`  | Implemented system-related commands `restart`, `free`, `make`. |
| `ble_mesh_reg_cfg_client_cmd`| Implemented the configure client model related command `bmccm`. |
| `ble_mesh_reg_gen_onoff_client_cmd`| Implemented the onoff client model related command `bmgocm`.|
| `ble_mesh_reg_test_perf_client_cmd` | Implemented the performance test related commands `bmcperf`. |
| `ble_mesh_register_node_cmd`  | Implemented the node related commands `bmreg`, `bmoob`, `bminit`, `bmpbearer`, `bmtxpower`.  |
| `ble_mesh_register_provisioner_cmd` | Implemented the node provisioner commands `bmpreg`, `bmpdev`, `bmpbearer`, `bmpgetn`, `bmpaddn`,`bmpbind`,`bmpkey`. |
| `register_bluetooth`    | Implemented a command to get a Bluetooth address `btmac` |


# Example Walkthrough
## Main Entry Point
Initialize Bluetooth, then initialize the console, and finally register a series of commands.
```c
void app_main(void)
{
    esp_err_t res;
    nvs_flash_init();
    // init and enable bluetooth
    res = bluetooth_init();
    if (res) {
        printf("esp32_bluetooth_init failed (ret %d)", res);
    }
	
#if CONFIG_STORE_HISTORY
    initialize_filesystem();
#endif
    initialize_console();

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

    const char *prompt = LOG_COLOR_I "esp32> " LOG_RESET_COLOR;

    /* Figure out if the terminal supports escape sequences */
    int probe_status = linenoiseProbe();
    if (probe_status) { /* zero indicates OK */
        printf("\n"
               "Your terminal application does not support escape sequences.\n"
               "Line editing and history features are disabled.\n"
               "On Windows, try using Putty instead.\n");
        linenoiseSetDumbMode(1);
#if CONFIG_LOG_COLORS
        /* Since the terminal doesn't support escape sequences,
         * don't use color codes in the prompt.
         */
        prompt = "esp32> ";
#endif //CONFIG_LOG_COLORS
    }
    
```

##  Main Program

The main program constantly reads data from the command line.`esp_console_run` will parse the argument and call the callback function of the previously initialized command.
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



