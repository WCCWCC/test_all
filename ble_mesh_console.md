# 1. Introduction
## 1.1 Demo Function

1. This is a demo that wifi and bluetooth coexist.You can use the wifi function while operating Bluetooth.
2. wifi function in this demo: Testing the transfer rate of wifi by `iperf`.
3. bluetooth function in this demo:The function of Bluetooth is the function of `ble_mesh_fast_prov_server`.

Noet:In this demo,you call wifi API and bluetooth API.such as `wifi_get_local_ip` and `esp_ble_mesh_provisioner_add_unprov_dev`.

# 2. How to use this demo
You need to download the project(`ble_mesh_wifi_coexist`) code to board.
Enter the following command by connecting to the borad terminal：
1. run `sta ssid password` in board terminal.
If you connect to the wifi named `tset_wifi` and the wifi password `12345678`.You should enter the command `sta tset_wifi 12345678`

2. run `iperf -s -i 3 -t 1000` in board terminal.
This command starts a tcp server to test the transfer rate of wifi.

3. run `iperf -c 192.168.10.42 -i 3 -t 60` in PC terminal.

4. You can use Bluetooth at the same time.Control node light switch.

The log:
```c
esp32> iperf -s -i 3 -t 1000
I (31091) iperf: mode=tcp-server sip=192.168.43.239:5001, dip=0.0.0.0:5001, interval=3, time=1000

        Interval Bandwidth
esp32>    0-   3 sec       0.00 Mbits/sec
   3-   6 sec       0.00 Mbits/sec
   6-   9 sec       0.00 Mbits/sec
accept: 192.168.43.100,60346
   9-  12 sec       0.04 Mbits/sec
  12-  15 sec       0.22 Mbits/sec
  15-  18 sec       0.20 Mbits/sec
  18-  21 sec       0.01 Mbits/sec
  21-  24 sec       0.14 Mbits/sec
  24-  27 sec       0.06 Mbits/sec
  27-  30 sec       0.07 Mbits/sec
  30-  33 sec       0.20 Mbits/sec
  33-  36 sec       0.17 Mbits/sec
  36-  39 sec       0.36 Mbits/sec
  39-  42 sec       0.18 Mbits/sec
  42-  45 sec       0.18 Mbits/sec
  45-  48 sec       0.38 Mbits/sec
  48-  51 sec       0.18 Mbits/sec
  51-  54 sec       0.46 Mbits/sec
  54-  57 sec       0.45 Mbits/sec
  57-  60 sec       0.16 Mbits/sec
  60-  63 sec       0.33 Mbits/sec
```

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



demo 描述一下：
实现node + proovisioner

实现 message 发送和接收





