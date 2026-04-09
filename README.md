| Supported Targets | ESP32-H2 | 
| ----------------- | -------- | 

ESP32 Root Network Module
==================================
## Table of Contents
- [ESP32 Root Network Module](#esp32-root-network-module)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Hardware Components](#hardware-components)
  - [Software Components](#software-components)
  - [Setup and Configuration](#setup-and-configuration)
    - [1. Downloading ESP-IDF Extension on VSCode (*Recommended*)](#1-downloading-esp-idf-extension-on-vscode-recommended)
    - [2. Using Docker Images on ESP-IDF](#2-using-docker-images-on-esp-idf)
  - [Communication Protocols](#communication-protocols)
  - [Code Structure](#code-structure)
  - [Code Flow](#code-flow)
    - [1) Initialization](#1-initialization)
    - [2) UART Channel Logic Flow](#2-uart-channel-logic-flow)
    - [3) Network Commands - UART incoming](#3-network-commands---uart-incoming)
    - [4) Module to App level - UART outgoing](#4-module-to-app-level---uart-outgoing)
    - [5) Event Handler](#5-event-handler)
    - [Error Handling](#error-handling)
  - [Testing and Troubleshooting](#testing-and-troubleshooting)
  - [References](#references)

## Overview
The ESP32 Root Module serves as the provisioner and central gateway of the BLE mesh network. It is responsible for provisioning edge nodes, maintaining network state, relaying UART commands from the host application into the mesh, and forwarding data received from edge nodes back to the host application.

The root module currently supports the following major features:

- **Network Gateway**
  - Receives UART commands from the host application
  - Sends BLE mesh messages to edge nodes
  - Forwards incoming edge data back to the host over UART

- **Provisioner**
  - Acts as the main provisioner for the BLE mesh network
  - Stores and tracks provisioned node information
  - Supports remote provisioning behavior through the BLE mesh stack

- **Network Introspection**
  - Reports provisioned node information (`NINFO`)
  - Reports directed forwarding table entries (`GETDF`)

- **Reliability / Observability**
  - Sends an ACK back to the edge when a message is received
  - Helps edge nodes measure round-trip time (RTT)
  - Logs whether traffic was received through Directed Forwarding or Flooding

- **Application Data Forwarding**
  - Receives GPS payloads from edge nodes
  - Converts GPS structs into JSON and forwards them to the Python server / host application over UART
      
## Hardware Components
For more information please contact the author if interested on the Custom PCB or Antenna.

## Software Components
- ESP-IDF version 5.2.0 (Espressif IoT Development Framework)
  - Description: Official development framework for ESP32
  - Function: Provides libraries and tools for developing applications on the ESP32
    - Build, Flash, Monitor, etc.
  - Instalation: [link to ESP's website](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html)

## Setup and Configuration
In this section, we will be explaining 2 ways on using our program, specifically ESP-IDF.

### 1. Downloading ESP-IDF Extension on VSCode (*Recommended*)
- Make sure you have [VS Code](https://code.visualstudio.com/download), it can be any operating system, or any version of VS Code.
- The next step is to download ESP-IDF Extension on VSCode. There are steps
on using ESP-IDF Extension in this [link](https://github.com/espressif/vscode-esp-idf-extension/blob/master/docs/tutorial/install.md)
- **P.S. Make sure the ESP-IDF version is 5.2.0, without this version, our code would not be able to run.**
- Once you have follow the steps on installing ESP-IDF, you are ready to `build`, `flash`, and `monitor`.
- **P.S. If you are on windows, you need to install a driver to establish a serial connection with the ESP32 Board. You can find it on this [website](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/establish-serial-connection.html). The author use the `CP210x USB to UART Bridge Drivers` to connect the windows port to the ESP32.**

### 2. Using Docker Images on ESP-IDF
- If you don't want to download ESP-IDF Extension, you can also use a Docker Image to `build` and `flash` the program. However, this only works in `Linux` system since you need a port number that's connected to the ESP32 when `flashing`
- The steps are as follows:
  1. Make sure you have [Docker Desktop](https://www.docker.com/products/docker-desktop/) downloaded and running in the background 
  2. Go to the terminal, and go to the project directory. (If you want to make sure you are in your project directory, you can write `${PWD}`, if this returns the project directory, that means you're in the right palce)
  3. Run `docker run --rm -v ${PWD}:/project -w /project -e HOME=/tmp espressif/idf:v5.2 idf.py build`
  4. Once it's done building, then you can run `docker run --rm -v ${PWD}:/project -w /project -e HOME=/tmp espressif/idf:v5.2 idf.py -p -PORT flash`. to flash to a ESP32 Board

  The `-PORT` you can change it to the port you're ESP32 is connected to, for example `/dev/ttyS5`. For more information on the docker image ESP32, click [here](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-docker-image.html)

  Also, after you're done flashing, you could also write `idf.py monitor --no-reset -p -PORT`. For more information, you can check [here](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-monitor.html)
  
## Communication Protocols

### UART Incoming Commands (Host -> Root)
The root module accepts fixed-length 5-byte command headers followed by command-specific payloads.

Supported commands:

- `NINFO`
  - Requests the current list of provisioned nodes
  - Root replies with one or more UART packets containing node address + UUID batches

- `GETDF`
  - Requests the current directed forwarding table stored on the root
  - Root replies with one or more UART packets containing `(origin, target)` path pairs

- `SEND-`
  - Sends a unicast message into the mesh
  - Payload format:
    - `5-byte command`
    - `2-byte destination address` (network byte order)
    - `message payload`

- `BCAST`
  - Sends a broadcast message into the mesh
  - Payload format:
    - `5-byte command`
    - `2-byte reserved address field`
    - `message payload`

- `RST-R`
  - Restarts the root module

- `CLEAN`
  - Clears persistent network configuration / mesh state

### UART Outgoing Messages (Root -> Host)

The root forwards different kinds of data back to the host application.

#### 1. Normal mesh payload forwarding
For regular data received from a node, the UART payload format is:

`2-byte node_addr | payload`

where `node_addr` is in network byte order.

#### 2. Provisioned node information
For `NINFO`, the root sends batched node information packets:

- `0x01 | batch_size | repeated(2-byte node_addr + 16-byte uuid)`

The root also sends node-specific configuration information using:

- `0x02 | 16-byte uuid`

during config completion handling.

#### 3. Directed forwarding table information
For `GETDF`, the root sends:

- `0x04 | batch_size | repeated(2-byte origin + 2-byte target)`

If no DF paths exist, it sends:

- `0x04 | 0x00`

#### 4. GPS JSON forwarding
If the root receives a BLE message whose payload size matches `gps_data_t`, it parses the struct and forwards it to UART as JSON. The JSON includes fields such as:

- `node`
- `gps_time`
- `lat`
- `lon`
- `fixType`
- `gnssFixOK`
- `diffSoln`
- `numSV`
- `button`

## Code Structure
This repository contains several files and directories, but the most important ones for the root module are listed below:

- **`/main`**
  - **`ble_mesh_config_root.c`**  
    Contains root-side BLE mesh initialization, provisioning logic, directed forwarding callbacks, and root utility APIs.
  - **`ble_mesh_config_root.h`**  
    Root-side declarations, data structures, and shared interfaces.
  - **`board.c`**  
    Hardware-related functions such as UART helpers and board-level support.
  - **`board.h`**
  - **`main.c`**  
    Root-side application logic, UART command parsing, and high-level event handlers.
  - **`idf_component.yml`**
  - **`CMakeLists.txt`**

- **`/Secret`**
  - Mesh network configuration headers such as app key / network definitions.

- **Top-level build files**
  - **`CMakeLists.txt`**
  - **`sdkconfig.defaults`**

## Code Flow
### 1) Initialization
When the root module boots, `app_main()` initializes the BLE mesh root stack, board-level peripherals, and the UART receive task. The root attaches application-level callback handlers for provisioning, configuration completion, message reception, response reception, timeout handling, broadcast reception, and connectivity checks.

The BLE mesh root initialization is performed through `esp_module_root_init(...)`, which:
- validates callback pointers
- initializes NVS
- initializes Bluetooth
- binds the AppKey to local vendor models
- initializes the BLE mesh subsystem
- prepares important-message tracking storage

After initialization, the root starts the UART receive task so that host commands such as `NINFO`, `GETDF`, `SEND-`, `BCAST`, `RST-R`, and `CLEAN` can be processed.

```c
void app_main(void)
{
    esp_log_level_set(TAG_ALL, ESP_LOG_NONE); // disable esp logs
    
    esp_err_t err = esp_module_root_init(prov_complete_handler, config_complete_handler, recv_message_handler,       
                              recv_response_handler, timeout_handler, broadcast_handler, connectivity_handler);
    if (err != ESP_OK) {
        ESP_LOGE(TAG_M, "Network Module Initialization failed (err %d)", err);
        uart_sendMsg(0, "Error: Network Module Initialization failed\n");
        return;
    }

    board_init();
    xTaskCreate(rx_task, "uart_rx_task", 1024 * 2, NULL, configMAX_PRIORITIES - 1, NULL);

    char message[15] = "online\n";
    uint8_t message_byte[15];
    message_byte[0] = 0x03; // Root Reset
    memcpy(message_byte + 1, message, strlen(message));
    uart_sendData(0, message_byte, strlen(message) + 1);
    printNetworkInfo(); // esp log for debug
}
```
In `line 5`, `esp_module_root_init` is called to initialize the ESP Module which includes multiple functions that are used as callback functions for Network events.
```c
esp_err_t esp_module_root_init(
    void (*prov_complete_handler)(uint16_t node_index, const esp_ble_mesh_octet16_t uuid, 
                                  uint16_t addr, uint8_t element_num, uint16_t net_idx),
    void (*config_complete_handler)(uint16_t addr),
    void (*recv_message_handler)(esp_ble_mesh_msg_ctx_t *ctx, uint16_t length, uint8_t *msg_ptr),
    void (*recv_response_handler)(esp_ble_mesh_msg_ctx_t *ctx, uint16_t length, uint8_t *msg_ptr),
    void (*timeout_handler)(esp_ble_mesh_msg_ctx_t *ctx, uint32_t opcode),
    void (*broadcast_handler)(esp_ble_mesh_msg_ctx_t *ctx, uint16_t length, uint8_t *msg_ptr),
    void (*connectivity_handler)(esp_ble_mesh_msg_ctx_t *ctx, uint16_t length, uint8_t *msg_ptr)
) { ... }
```
Each handler function will get trigers by corresponding event [link here](#event-handler)

### 2) Message Reception, ACK, and GPS Forwarding
When the root receives a mesh message from an edge node, `recv_message_handler(...)` first sends an ACK back to the sender using `ECS_193_MODEL_OP_RESPONSE`.

This ACK is used by the edge node to measure RTT on the edge side.

After that, the root:
- logs the source node address and opcode
- checks whether the message was received via Directed Forwarding or Flooding
- checks whether the payload is a GPS struct

If the payload size matches `gps_data_t`, the root interprets the message as GPS data, extracts the fields, and serializes them into JSON for forwarding over UART to the Python server / host application.

Otherwise, the payload is forwarded over UART using the standard:

`2-byte node_addr | payload`

format.

### 3) UART Channel Logic Flow
The module communicate with central PC via usb-uart port.

1. `UART byte encoding` - To ensure the message bytes' integrity, message encoding was applied to add `\0xFF` and `\0xFE` speical bytes at the begining and end of an uart message. Also the message byte encoding was applied to encode all bytes >= `\0xFA` into 2 byte with xor gate to reserve all bytes > `\0xFA` as speical bytes. Uart encoding, decoding, and write functions is defined in `board.h` file with detailed explainatiion.

2. `UART channel listening thread` - The function `rx_task()` on main.c defines the uart signal handling logic. It create an infinite scanning loop to check uart buffer's data avalaibility. Once the scanner read in datas, it passes to `uart_task_handler()` to scan for message start byte `\0xFF` and message end byte `\0xFE` to locate the message then decode the messsage and invoke `execute_uart_command()` to parse and execute the message received.

3. `execute_uart_command()` - This function responsible for executing commands from application level such as `BCAST`, `SEND-`, and etc. The module able to be extended for custom command by adding a case in this function.

### 4) Network Commands - UART incoming
The formate of network commands send to esp module is defined to consist `5_byte_network_command | payload` where the payloadi's format varys based on the command and detils on current commands is documented here. `[Add link later]------------------------`

### 5) Module to App level - UART outgoing
The formate of esp module to app level message is defined as `2_byte_node_addr | payload`. The first part is `netword endian` encoding of address of the node associated with the payload. For instance, the main use case is when module recived and message from src node `5`; the uart message will be `0x00 0x05 | message from node 5` (the uart escape byte endoing still get applied on top of this). 

Other use cases are module's own address for module status message or debug use to pass 2 byte critical informations.

### 6) Event Handler
The root module uses callback-based event handlers so that BLE mesh stack logic remains in `ble_mesh_config_root.c`, while higher-level application behavior stays in `main.c`.

The main handlers are:

- `prov_complete_handler`
  - Invoked when a node has been provisioned

- `config_complete_handler`
  - Invoked when a node has completed configuration
  - Sends node information to the host application

- `recv_message_handler`
  - Invoked when the root receives a message from a node
  - Sends an ACK back to the edge
  - Detects whether the message arrived through Directed Forwarding or Flooding
  - Detects GPS payloads and forwards them as JSON over UART

- `recv_response_handler`
  - Invoked when the root receives a response to a response-expected message

- `timeout_handler`
  - Invoked when a response-expected message times out
  - Used for important-message retransmission logic

- `broadcast_handler`
  - Invoked when the root receives a broadcast message

- `connectivity_handler`
  - Invoked when the root receives a connectivity / heartbeat message

### Error Handling
Known issues / considerations include:

- **UART framing / partial message issue**
  - UART messages are delimited using special start/end bytes.
  - If only part of a framed message is read into the buffer, the code may report a leftover / half-message warning.

- **Persistent mesh state mismatch**
  - If the root resets or clears persistent state while edge nodes still keep old provisioning state, reprovisioning inconsistencies may occur.
  - In those cases, `CLEAN` / reset workflows may be required.

- **Wrong UART port / host contention**
  - The root UART is shared with the external host application logic.
  - Incorrect serial port configuration or multiple host-side readers can cause communication failures.

- **Directed forwarding visibility**
  - The root can report whether a message was received via Directed Forwarding or Flooding, but application-level testing should still validate expected path behavior.


OPTIONAL:
potencial error and warning and current fix.

## Testing and Troubleshooting

### Suggested tests
- Send `NINFO` from the host and verify the root returns provisioned node batches.
- Send `GETDF` from the host and verify the root returns directed forwarding path entries.
- Send a normal unicast payload using `SEND-` and verify:
  - the edge receives it
  - the root can still process ACK / response traffic correctly
- Send GPS data from an edge node and verify the root forwards JSON over UART.
- Observe root logs to confirm whether received traffic used Directed Forwarding or Flooding.

### Common troubleshooting tips
- Make sure the correct serial port is selected before flashing or monitoring.
- Avoid multiple host-side processes reading the same UART port.
- If the network state looks inconsistent, reset / clean mesh state and reprovision nodes.

## References
[ESP_BLE_MESH](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/esp-ble-mesh/ble-mesh-index.html)

[ESP_BLE_MESH Github Project Examples](https://github.com/espressif/esp-idf/blob/master/examples/bluetooth/esp_ble_mesh/README.md)