# 1. Introduction
## 1.1 Demo Function

1. This is a demo that help you quickly implement BLE MESH related functions
2. You can implement the provisioner to control the device to access the mesh network.
3. You can implement node to control the state of the light.

Noet:In this demo,You can use the `help` command to see all the commands currently supported.You can also modify the code and add the commands you want.

# 2. How to use this demo
You need to download the project(`ble_mesh_console/ble_mesh_provisioner`) code to board.

## 2.1 Implement a provisioner

## 2.2 Implement a node


# 3. Project Structure
The folder `ble_mesh_provisioner` contains the following files and subfolders:

| File Name        |Description               |
| ----------------------|------------------------- |
| `ble_mesh_adapter`      | The message opcode  |
| `ble_mesh_cfg_srv_model`       | The pointer to the client model struct  |
| `ble_mesh_console_lib` | The NetKey Index of the subnet through which the message is sent |
| `ble_mesh_console_main` | The AppKey Index for message encryption |
| `ble_mesh_console_system`    | The address of the destination nodes |
| `ble_mesh_reg_cfg_client_cmd`| The TTL State, which determines how many times a message will be relayed |
| `ble_mesh_reg_gen_onoff_client_cmd`| This parameter determines if the model will wait for an acknowledgement after sending a message   |
| `ble_mesh_reg_test_perf_client_cmd` | The maximum time the model will wait for an acknowledgement   |
| `ble_mesh_register_node_cmd`    | The role of message (node/provisioner)  |
| `ble_mesh_register_provisioner_cmd`    | The role of message (node/provisioner)  |
| `register_bluetooth`    | The role of message (node/provisioner)  |




# Example Walkthrough
## Main Entry Point
The program’s entry point is the app_main() function:

Initialize bluetooth related functions, then initialize the wifi console.

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
	bmreg

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

	esp_ble_mesh_node_prov_enable


    /* Prompt to be printed before each line.
     * This can be customized, made dynamic, etc.
     */
    const char *prompt = LOG_COLOR_I "esp32> " LOG_RESET_COLOR;

    printf("\n"
           "This is an example of an ESP-IDF console component.\n"
           "Type 'help' to get the list of commands.\n"
           "Use UP/DOWN arrows to navigate through the command history.\n"
           "Press TAB when typing a command name to auto-complete.\n");

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
#if CONFIG_STORE_HISTORY
        /* Save command history to filesystem */
        linenoiseHistorySave(HISTORY_PATH);
#endif

        /* Try to run the command */
        int ret;
        esp_err_t err = esp_console_run(line, &ret);
        if (err == ESP_ERR_NOT_FOUND) {
            printf("Unrecognized command\n");
        } else if (err == ESP_ERR_INVALID_ARG) {
            // command was empty
        } else if (err == ESP_OK && ret != ESP_OK) {
            printf("\nCommand returned non-zero error code: 0x%x\n", ret);
        } else if (err != ESP_OK) {
            printf("Internal error: 0x%x\n", err);
        }
        /* linenoise allocates line buffer on the heap, so need to free it */
        linenoiseFree(line);
    }
}

```

## wifi_console_init
`wifi_console_init` starts by initializing basic functions of wifi.
```c
    initialise_wifi();
    initialize_console();

    /* Register commands */
    esp_console_register_help_command();
    register_wifi();
```
`initialise_wifi` is used to set the working mode of wifi.
1. Set current WiFi power save type `WIFI_PS_MIN_MODEM`.In this mode, station wakes up to receive beacon every DTIM period
2. Set the WiFi API configuration storage type `WIFI_STORAGE_RAM`. all configuration will only store in the memory
3. Set the WiFi operating mode `WIFI_MODE_STA`. Wifi will work in station mode.

Wifi is used by console command line.You can view the currently supported wifi commands via the input `help` command.
`register_wifi` registered the following commands: `sta`,`scan`,`ap`,`query`,`iperf`,`restart`,`heap`.
For example: register `start` console command.The handler for this command is `restart`.The `restart` will run while you input command `restart`.
```c
    static int restart(int argc, char **argv)
    {
        ESP_LOGI(TAG, "Restarting");
        esp_restart();
    }
    const esp_console_cmd_t restart_cmd = {
        .command = "restart",
        .help = "Restart the program",
        .hint = NULL,
        .func = &restart,
    };
```


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



