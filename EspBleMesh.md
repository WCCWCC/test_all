# Demo Function

This demo demonstrates the fast provisioning of ESP BLE Mesh network and how to use the EspBleMesh app to control an individual provisioned node or all the provisioned nodes.
 
A video of this demo can be seen ![here](http://download.espressif.com/BLE_MESH/BLE_Mesh_Demo/V0.4_Demo_Fast_Provision/ESP32_BLE_Mesh_Fast_Provision.mp4)

# What You Need 

* ![EspBleMesh App for Android](http://download.espressif.com/BLE_MESH/ESP_BLE_MESH_APKs/EspBleMesh-0.9.3.apk)
* ![ESP BLE Mesh SDK v0.6(Beta Version
d)](https://glab.espressif.cn/ble_mesh/esp-ble-mesh-v0.6)
* ESP32 Development Boards

> Note:
> 
> 1. Please flash the ![`ble_mesh_fast_prov_server`（改为相对路径）](examples/bluetooth/ble_mesh/ble_mesh_fast_provision/ble_mesh_fast_prov_server) to your boards first;
> 2. To have a better understanding of the performance of the BLE Mesh network, we recommend that at least 3 devices should be added in your network.
> 3. We recommend that you solder LED indicators if your development board does not come with lights. 
> 4. Please check the type of board and LED pin definition enabled in `Example BLE Mesh Config` by running `make menuconfig`
  
Figure 1: Board

# Flash and Monitor

1.	Enter the directory: 
examples/bluetooth/ble_mesh/ble_mesh_fast_provision/ble_mesh_fast_prov_server
2.	Make sure that the `IDF_PATH` environment variable was set in accordance with your current IDF path
3. Check the version of your toolchain. Version 4.1 or newer should be used.
 
 
Figure 2: Check environment

4. Run `make -j4 flash` to compile codes and flash the codes to the device.
 
Figure 3: compiled code

> Note: 
> 
> Please click on the Exit button if you see the following windows.
                   
Figure 4:Click exit.

5. Please establish a connection between your device and PC, using the correct serial number, if you want to monitor the operation of this device on PC. 

# How to Use the App

Please launch the `EspBleMesh` app, and follow the steps described below to establish a BLE Mesh network and control any individual node or all the nodes.
 
 
1. Click on the upper left corner to see more options;
2. Click on **Provisioning** to scan nearby unprovisioned devices;
3. Choose any unprovisioned devices in the scanned list;
4. Enter the number of devices you want to add in your mesh network;
5. Wait until all the devices are provisioned;
6. Click on the upper left corner to see more options;
7. Click on **Fast Provisioned** to see all the provisioned devices;
8. Control your devices.

Figure 5: App steps


> Note: 
> 
> Please disable your Bluetooth function on your phone, enable it and try again, if you have encountered any connection issues.


# Procedure

## Role

* Phone - Top Provisioner
* The device that has been provisioned by Phone - Primary Provisioner
* Devices that have been provisioned and changed to the role of a provisioner - Temporary Provisioner
* Devices that have been provisioned but not changed to the role of a provisioner - Node

## Interaction

1. The Top Provisioner configures the first device to access the network with the GATT bearer.
2. The Top Provisioner sends the `send_config_appkey_add` message to allocate the Appkey to this device. 
3. The Top Provisioner sends the `send_fast_prov_info_set` message to provide the necessary information so the device can be changed to a Primary Provisioner.
4. The device calls the `esp_ble_mesh_set_fast_prov_action` API to change itself into the role of a Primary Provisioner and disconnects with the Top Provisioner.
5. The Primary Provisioner sends the `send_config_appkey_add` message to allocate the Appkey to an other device.
6. The Primary Provisioner sends the `send_fast_prov_info_set` message to provide the necessary information so the device can be changed to a Temporary Provisioner.
7. The device calls the `esp_ble_mesh_set_fast_prov_action` API to change itself into the role of a Temporary Provisioner and starts its address timer.
8. The Temporary Provisioner collects the addresses of nodes that it has provisioned and sends these addresses to the Primary Provisioner, when its address timer times out, which indicates the Temporary Provisioner hasn't provisioned any devices for xxx.
9. The Primary Provisioner reconnects to the Top Provisioner when its xxx timer times out, which indicates the Primary Provisioner hasn't received any messages from the Temporary Provisioners for xxx.
10. The Top Provisioner sends the `the node address Get` message automatically after reconnecting with the Primary Provisioner.
11. At this point, the Top Provisioner is able to control any nodes in the BLE Mesh Network.


> Note:
> 
> The nodes in the BLE Mesh Network only disable its provisioner functionality after it has been controlled by the Top Provisioner for at least one time. 







Using the EspBleMesh App, click Provisioning. The role of the phone at this time is Top Provisioner.

2. Choose any of the scanned device in the list and enter the number of devices you need to access. After clicking ok, the node will be configured to access the network.

3. After the node enters the network, Top Provisioner will send the message send_config_appkey_add through [config client model] to add appkey to the [config server model] node.

4. Then Top Provisioner will send another message send_fast_prov_info_set to the [vender server model] of the node via [vender client model]. 
The node will get the fast provisioning information and call the esp_ble_mesh_set_fast_prov_action API to enable the provisioner functionality. The role of the node is Primary Provisioner.

5. At this point, the Primary Provisioner will disconnect from the Top Provisioner.

6. Primary Provisioner will provision other unprovisioned devices (3 at most). After the device is provisioned into the network, the Primary Provisioner will configure it properly and start the Temporary Provisioner functionality.

7. After this the Temporary Provisioner will start a timer. When the timer expires, it will report the addresses of the nodes, which are provisioned by the Temporary Provisioner, to the Primary Provisioner by sending the Fast Provisioning Node Address message. When The Primary Provisioner receives the message, it will send a Fast Provisioning Node Address ACK message. Temporary Provisioner will be equipped with other un-allocated devices.

8.After receiving the first address message reported by the Temporary Provisioner, the Primary Provisioner will start a timer. When the timer expires, the connection is established again with the Top Provisioner (Phone). User can click the “GET” button in the menu to get all the node addresses.

9. Top Provisioner obtains the address o
10. f all nodes by sending Node_Address_Get, so that each device can be controlled separately.

10. Top Provisioner sends an onoff_control message with the destination address being the group address. All nodes that receive the message will call the esp_ble_mesh_set_fast_prov_action API to disable the function of the fast distribution network.

 

Figure 6: Interaction logic
