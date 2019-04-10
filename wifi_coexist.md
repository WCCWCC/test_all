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
