# mxchip-az3166

## <b>Nx_MQTT_Client Application Description</b>

This application provides an example of Azure RTOS NetX/NetXDuo stack usage .

It shows how to exchange data between client and server using MQTT protocol in an encrypted mode supporting TLS v1.2.

The main entry function tx_application_define() is called by ThreadX during kernel start, at this stage, all NetX resources are created.

- A <i>NX_PACKET_POOL</i>is allocated

- A <i>NX_IP</i> instance using that pool is initialized

- The <i>ARP</i>, <i>ICMP</i>, <i>UDP</i> and <i>TCP</i> protocols are enabled for the <i>NX_IP</i> instance

- A <i>DHCP client is created.</i>

The application then creates 3 threads with the different priorities:

- **AppMainThread** (priority 10, PreemtionThreashold 10) : created with the <i>TX_AUTO_START</i> flag to start automatically.

- **AppMQTTClientThread** (priority 10, PreemtionThreashold 10) : created with the <i>TX_DONT_START</i> flag to be started later.

The **AppMainThread** starts and perform the following actions:

- Starts the DHCP client

- Waits for the IP address resolution

- Resumes the **AppSNTPThread**

The **AppSNTPThread**, once started:

- it connects to an SNTP server to get current time in order to check x509 certificate validation then it resumes **AppMQTTClientThread**.

The **AppMQTTClientThread**, once started:

- creates a dns_client with USER_DNS_ADDRESS used as DNS server then get MQTT broker address from MQTT_BROKER_NAME predefined on app_netxduo.h.

- creates an mqtt_client

- connects mqtt_client to the online MQTT broker; connection with server will be secure and a **tls_setup_callback** will set TLS parametres. By default MQTT_PORT for encrypted mode is 8883.

        refer to note 1 below, to know how to setup an x509 certificate.

- mqtt_client subscribes on a predefined topic TOPIC_NAME "Temperature" with a Quality Of Service QOS0

- depending on the number of message "NB_MESSAGE" defined by user, mqtt_client will publish a random number generated by RNG. If NB_MESSAGE = 0, it means that number of messages is infinitely.

- mqtt_client will get messages from the MQTT broker and print them.

#### <b>Expected success behavior</b>

- The board IP address is printed on the HyperTerminal

- Connection's information are printed on the HyperTerminal (broker's name, mqtt port, topic and messages received)

```
MQTT client connected to broker < test.mosquitto.org > at PORT 8883 :
Message 1 received: TOPIC = Temperature, MESSAGE = 34
Message 2 received: TOPIC = Temperature, MESSAGE = 35
Message 3 received: TOPIC = Temperature, MESSAGE = 35
Message 4 received: TOPIC = Temperature, MESSAGE = 34
Message 5 received: TOPIC = Temperature, MESSAGE = 33
client disconnected from server
```

- Green led is toggling after successfully receiving all messages.

#### <b>Error behaviors</b>

- The red LED is toggling to indicate any error that has occurred.

#### <b>Assumptions if any</b>

None

#### <b>Known limitations</b>

- Topic name length is by default =< NXD_MQTT_MAX_TOPIC_NAME_LENGTH = 12, so user should adjust this value manually (on nxd_mqtt_client.h) if his topic name length will exceed the default value.

- Message length is by default =< NXD_MQTT_MAX_MESSAGE_LENGTH = 32, so user should adjust this value manually (on nxd_mqtt_client.h) if his message length will exceed the default value.

### <b>Notes</b>

1.  To make an encrypted connection with MQTT server, user should follow these steps to add an x509 certificate to the _mqtt_client_ and use it to ensure server's authentication :

    - download certificate authority CA (in this application "mosquitto.org.der" downloaded from [test.mosquitto](https://test.mosquitto.org)

    - convert certificate downloaded by executing the following cmd from the file downloaded path :

                    xxd.exe -i mosquitto.org.der > mosquitto.cert.h

    - add the converted file under the application : NetXDuo/Nx_MQTT_Client/NetXDuo/App

    - configure MOSQUITTO_CERT_FILE with your certificate name.

#### <b>ThreadX usage hints</b>

- ThreadX uses the Systick as time base, thus it is mandatory that the HAL uses a separate time base through the TIM IPs.

- ThreadX is configured with 100 ticks/sec by default, this should be taken into account when using delays or timeouts at application. It is always possible to reconfigure it in the "tx_user.h", the "TX_TIMER_TICKS_PER_SECOND" define,but this should be reflected in "tx_initialize_low_level.s" file too.

- ThreadX is disabling all interrupts during kernel start-up to avoid any unexpected behavior, therefore all system related calls (HAL, BSP) should be done either at the beginning of the application or inside the thread entry functions.

- ThreadX offers the "tx_application_define()" function, that is automatically called by the tx_kernel_enter() API.
  It is highly recommended to use it to create all applications ThreadX related resources (threads, semaphores, memory pools...) but it should not in any way contain a system API call (HAL or BSP).

- Using dynamic memory allocation requires to apply some changes to the linker file.

  ThreadX needs to pass a pointer to the first free memory location in RAM to the tx_application_define() function, using the "first_unused_memory" argument.
  This require changes in the linker files to expose this memory location.

  - For EWARM add the following section into the .icf file:

  ```
  place in RAM_region    { last section FREE_MEM };
  ```

  - For MDK-ARM:

  ```
   either define the RW_IRAM1 region in the ".sct" file
   or modify the line below in "tx_low_level_initilize.s to match the memory region being used
       LDR r1, =|Image$$RW_IRAM1$$ZI$$Limit|
  ```

  - For STM32CubeIDE add the following section into the .ld file:

  ```
   ._threadx_heap :
     {
        . = ALIGN(8);
        __RAM_segment_used_end__ = .;
        . = . + 64K;
        . = ALIGN(8);
      } >RAM_D1 AT> RAM_D1
  ```

      The simplest way to provide memory for ThreadX is to define a new section, see ._threadx_heap above.
      In the example above the ThreadX heap size is set to 64KBytes.
      The ._threadx_heap must be located between the .bss and the ._user_heap_stack sections in the linker script.
      Caution: Make sure that ThreadX does not need more than the provided heap memory (64KBytes in this example).
      Read more in STM32CubeIDE User Guide, chapter: "Linker script".

  - The "tx_initialize_low_level.s" should be also modified to enable the "USE_DYNAMIC_MEMORY_ALLOCATION" flag.

### <b>Keywords</b>

RTOS, Network, ThreadX, NetXDuo, WIFI, MQTT, DNS, TLS, MXCHIP, UART

### <b>Hardware and Software environment</b>

- This example runs on STM32U585xx devices with a WiFi module (MXCHIP:EMW3080) with this configuration :

  - MXCHIP Firmware 2.1.11

  - Bypass mode

  - SPI mode used as interface

  The EMW3080B MXCHIP Wi-Fi module firmware and the way to update your board with it are available at <https://www.st.com/en/development-tools/x-wifi-emw3080b.html>

  Before using this project, you shall update your B-U585I-IOT02A board with the EMW3080B firmware version 2.1.11.

  To achieve this, follow the instructions given at the above link, using the EMW3080updateV2.1.11RevC.bin flasher under the V2.1.11/SPI folder.

- This application has been tested with B-U585I-IOT02A (MB1551-U585AI) boards Revision: RevC and can be easily tailored to any other supported device and development board.

- This application uses USART1 to display logs, the hyperterminal configuration is as follows:
  - BaudRate = 115200 baud
  - Word Length = 8 Bits
  - Stop Bit = 1
  - Parity = None
  - Flow control = None

### <b>How to use it ?</b>

In order to make the program work, you must do the following :

- Open your preferred toolchain

- On <code> Core/Inc/mx_wifi_conf.h </code> , Edit your Wifi Settings (WIFI_SSID, WIFI_PASSWORD)

- Edit the file app_netxduo.h : define the USER_DNS_ADDRESS, the MQTT_BROKER_NAME and NB_MESSAGE.

- Rebuild all files and load your image into target memory

- Run the application
