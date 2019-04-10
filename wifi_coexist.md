# 1. Introduction
## 1.1 Demo Function

This is a demo that wifi and bluetooth coexist.

Testing the transfer rate of wifi when Bluetooth is working properly.

The function of Bluetooth is the function of ble_mesh_fast_prov_server.

在ble_mesh_fast_prov_server例程的基础之上，添加了wifi的基本功能。
wifi通过命令行的方式，实现了哪些wifi的命令

重点是共存：1蓝牙功能  2wifi功能

# 2. How to use this demo
//在wifi高速传输的情况下，使用快速配网的功能。

Interval表示时间间隔,Bandwidth是时间间隔里的传输速率,TCP server , TCP  client.
//将关键的log粘贴出来

# 3. Project Structure
The folder `ble_mesh_node` contains the following files and subfolders:

```
$ tree examples/bluetooth/ble_mesh/ble_mesh/ble_mesh_node
├── Makefile    /* Compiling parameters for the demo */
├── README.md   /* Quick start guide */
├── build
├── main        /* Stores the `.c` and `.h` application code files for this demo */
├── sdkconfig      /* Current parameters of `make menuconfig` */
├── sdkconfig.defaults   /* Default parameters of `make menuconfig` */
├── sdkconfig.old      /* Previously saved parameters of `make menuconfig` */
└── tutorial         /* More in-depth information about the demo */
```

# Example Walkthrough

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
