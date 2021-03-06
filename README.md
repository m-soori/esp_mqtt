# esp_uMQTT_broker
An MQTT Broker/Client with scripting support on the ESP8266

This program enables the ESP8266 to become the central node in a small distributed IoT system. It implements an MQTT Broker and a simple scripted rule engine with event/action statements that links together the MQTT sensors and actors. It can act as STA, as AP, or as both and it can connect to another MQTT broker (i.e. in the cloud). Here it can act as bridge and forward and rewrite topics in both directions. Also it can write on local GPIO pins and react on GPIO interrupts.

Find a video that explains the ideas and the architecture of the project at: https://www.youtube.com/watch?v=0K9q4IuB_oA

# Usage
In the user directory there is the main program that serves as a stand-alone MQTT broker, client and bridge. The program starts with the following default configuration:

- ssid: ssid, password: password
- ap_ssid: MyAP, ap_password: none, ap_on: 1, ap_open: 1
- network: 192.168.4.0/24

This means it connects to the internet via AP ssid,password and offers an open AP with ap_ssid MyAP. This default can be changed in the file user_config.h. The default can be overwritten and persistenly saved to flash by using a console interface. This console is available either via the serial port at 115200 baud or via tcp port 7777 (e.g. "telnet 192.168.4.1 7777" from a connected STA).

Use the following commands for an initial setup:

- set ssid your_home_router's_SSID
- set password your_home_router's_password
- set ap_ssid ESP's_ssid
- set ap_password ESP's_password
- show (to check the parameters)
- save
- reset

After reboot it will connect to your home router and itself is ready for stations to connect.

The console understands the following commands:

General commands:

- help: prints a short help message
- show [config|stats]: prints the current config or some status information and statistics
- save: saves the current config parameters to flash
- lock [_password_]: saves and locks the current config, changes are not allowed. Password can be left open if already set before
- unlock _password_: unlocks the config, requires password from the lock command
- reset [factory]: resets the esp, 'factory' optionally resets WiFi params to default values (works on a locked device only from serial console)
- set speed [80|160]: sets the CPU clock frequency (default 80 Mhz)
- set config_port _portno_: sets the port number of the console login (default is 7777, 0 disables remote console config)
- set config_access _mode_: controls the networks that allow config access (0: no access, 1: only internal, 2: only external, 3: both (default))
- quit: terminates a remote session

WiFi and network related commands:

- set [ssid|password] _value_: changes the settings for the uplink AP (WiFi config of your home-rou- set [ap_ssid|ap_password] _value_: changes the settings for the soft-AP of the ESP (for your stations)ter)
- set ap_on [0|1]: selects, whether the soft-AP is disabled (ap_on=0) or enabled (ap_on=1, default)
- set ap_open [0|1]: selects, whether the soft-AP uses WPA2 security (ap_open=0,  automatic, if an ap_password is set) or open (ap_open=1)
- set auto_connect [0|1]: selects, whether the WiFi client should automatically retry to connect to the uplink AP (default: on=1)
- set network _ip-addr_: sets the IP address of the SoftAP's network, network is always /24, esp_uMQTT_broker is always x.x.x.1
- set dns _dns-addr_: sets a static DNS address
- set dns dhcp: configures use of the dynamic DNS address from DHCP, default
- set ip _ip-addr_: sets a static IP address for the ESP in the uplink network
- set ip dhcp: configures dynamic IP address for the ESP in the uplink network, default
- set netmask _netmask_: sets a static netmask for the uplink network
- set gw _gw-addr_: sets a static gateway address in the uplink network
- set mdns_mode [0|1|2]: selects, which interface should be announced via mDNS (0=none (default), 1 = STA, 2 = SoftAP)
- scan: does a scan for APs

While the user interface looks similar to my esp_wifi_repeater at https://github.com/martin-ger/esp_wifi_repeater this does NO NAT routing. AP and STA network are stricly separated and there is no routing in between. The only possible connection via both networks is the uMQTT broker that listens on both interfaces.

MQTT broker related command:

- show [mqtt]: prints the current config or status information of the MQTT broker
- set broker_user _unsername_: sets the username for authentication of MQTT clients ("none" if no auth, default)
- set broker_password _password_: sets the password for authentication of MQTT clients ("none" if empty, default)
- set broker_access _mode_: controls the networks that allow MQTT broker access (0: no access, 1: only internal, 2: only external, 3: both (default))
- set broker_subscriptions _max_: sets the max number of subscription the broker can store (default: 30)
- set broker_retained_messages _max_: sets the max number of retained messages the broker can store (default: 30)
- set script_logging [0|1]: switches logging of script execution on or off (not permanently stored in the configuration)
- script [_portno_|delete]: opens port for upload of scripts or deletes the current script
- set @[num] _value_: sets the flash variable "num" (for use in scripts) to the given inital value (must be shorter than 63 chars)

# MQTT client/bridging functionality
The broker comes with a "local" and a "remote" client, which means, the broker itself can publish and subscribe topics. The "local" client is a client to the own broker (without the need of an additional TCP connection).

By default the "remote" MQTT client is disabled. It can be enabled by setting the config parameter "mqtt_host" to a hostname different from "none". To configure the "remote" MQTT client you can set the following parameters:

- set mqtt_host _IP_or_hostname_: IP or hostname of the MQTT broker ("none" disables the MQTT client)
- set mqtt_user _username_: Username for authentication ("none" if no authentication is required at the broker)
- set mqtt_user _password_: Password for authentication
- set mqtt_id _clientId_: Id of the client at the broker (default: "ESPRouter_xxxxxx" derived from the MAC address)
- publish [local|remote] [topic] [data]: this publishes a topic (mainly for testing)

# Scripting
The esp_uMQTT_broker comes with a build-in scripting engine. A script enables the ESP not just to act as a passive broker but to react on events (publications and timing events) and to send out its own items.

Here is a demo of a script to give you an idea of the power of the scripting feature. This script controls a Sonoff switch module. It connects to a remote MQTT broker and in parallel offers locally its own. The device has a number stored in the variable $device_number. On both brokers it subscribes to a topic named '/martinshome/switch/($device_number)/command', where it receives commands, and it publishes the topic '/martinshome/switch/($device_number)/status' with the current state of the switch relay. It understands the commands 'on','off', 'toggle', and 'blink'. Blinking is realized via a timer event. Local status is stored in the two variables $relay_status and $blink (blinking on/off). The 'on gpio_interrupt' clause reacts on pressing the pushbutton of the Sonoff and simply toggles the switch (and stops blinking). The last two 'on clock' clauses implement a daily on and off period:

```
% Config params, overwrite any previous settings from the commandline
config ap_ssid 		MyAP
config ap_password	stupidPassword
config ntp_server	1.de.pool.ntp.org
config broker_user	Martin
config broker_password	secret
config mqtt_host	martinshome.fritz.box
config speed		160

% Now the initialization, this is done once after booting
on init
do
	% Device number
	setvar $device_number = 1

	% @<num> vars are stored in flash and are persistent even after reboot 
	setvar $run = @1 + 1
	setvar @1 = $run
	println "This is boot no "|$run

	% Status of the relay
	setvar $relay_status=0
	gpio_out 12 $relay_status
	gpio_out 13 not ($relay_status)

	% Blink flag
	setvar $blink=0

	% Command topic
	setvar $command_topic="/martinshome/switch/" | $device_number | "/command"

	% Status topic
	setvar $status_topic="/martinshome/switch/" | $device_number | "/status"

	publish local $status_topic $relay_status retained

	% local subscriptions once in 'init'
	subscribe local $command_topic

% Now the MQTT client init, this is done each time the client connects
on mqttconnect
do
	% remote subscriptions for each connection in 'mqttconnect'
	subscribe remote $command_topic

	publish remote $status_topic $relay_status retained

% Now the events, checked whenever something happens

% Is there a remote command?
on topic remote $command_topic
do
	println "Received remote command: " | $this_data

	% republish this locally - this does the action
	publish local $command_topic $this_data


% Is there a local command?
on topic local $command_topic
do
	println "Received local command: " | $this_data

	if $this_data = "on" then
		setvar $relay_status = 1
		setvar $blink = 0
		gpio_out 12 $relay_status
		gpio_out 13 not ($relay_status)
	else
	    if $this_data = "off" then
		setvar $relay_status = 0
		setvar $blink = 0
		gpio_out 12 $relay_status
		gpio_out 13 not ($relay_status)
	    endif
	endif
	if $this_data = "toggle" then
		setvar $relay_status = not ($relay_status)
		gpio_out 12 $relay_status
		gpio_out 13 not ($relay_status)
	endif
	if $this_data = "blink" then
		setvar $blink = 1
		settimer 1 500
	endif

	publish local $status_topic $relay_status retained
	publish remote $status_topic $relay_status retained


% The local pushbutton
on gpio_interrupt 0 pullup
do
	println "New state GPIO 0: " | $this_gpio
	if $this_gpio = 0 then
		setvar $blink = 0
		publish local $command_topic "toggle"
	endif


% Blinking
on timer 1
do
	if $blink = 1 then
		publish local $command_topic "toggle"
		settimer 1 500
	endif


% Switch on in the evening
on clock 19:30:00
do
	publish local $command_topic "on"

% Switch off at night
on clock 01:00:00
do
	publish local $command_topic "off"
```

Currently the interpreter is configured for a maximum of 10 variables, with a significant id length of 15. Some (additional) vars contain special status: $this_topic and $this_data are only defined in 'on topic' clauses and contain the current topic and its data. $this_gpio contains the state of the GPIO in an 'on gpio_interrupt' clause and $timestamp contains the current time of day in 'hh:mm:ss' format. If no NTP sync happened the time will be reported as "99:99:99". The variable "$weekday" returns the day of week as three letters ("Mon","Tue",...).

In general, scripts have the following BNF:

```
<statement> ::= on <event> do <action> |
		config <param> ([any ASCII]* | @<num>) |
                <statement> <statement>

<event> ::= init |
	    mqttconnect |
            timer <num> |
            clock <timestamp> |
            gpio_interrupt <num> (pullup|nopullup) |
            topic (local|remote) <topic-id>

<action> ::= publish (local|remote) <topic-id> <expr> [retained] |
             subscribe (local|remote) <topic-id> |
             unsubscribe (local|remote) <topic-id> |
             settimer <num> <expr> |
             setvar ($[any ASCII]* | @<NUM>) = <expr> |
             gpio_pinmode <num> [pullup]
             gpio_out <num> <expr> |
             if <expr> then <action> [else <action>] endif |
	     print <expr> | println <expr> |
	     system <expr> |
             <action> <action>

<expr> ::= <val> <op> <expr> | (<expr>) | not (<expr>)

<op> := '=' | '>' | gte | str_ge | str_gte | '+' | '-' | '*' | '|' | div

<val> := <string> | <const> | #<hex-string> | $[any ASCII]* | gpio_in(<num>) |
         @<num> | $this_item | $this_data | $this_gpio | $timestamp | $weekday

<string> := "[any ASCII]*" | [any ASCII]*

<num> := [0-9]*

<timestamp> := hh:mm:ss
```

Scripts with size up to 4KB are uploaded to the esp_uMQTT_broker using a network interface. Start the upload with "script <portno>" on the concole of the ESP, e.g.:
```
CMD>script 2000
Waiting for script upload on port 2000
CMD>

```
Now the ESP listens on the given port for an incoming connection and stores anything it receives as new script. Upload a file using netcat, e.g.:
```bash
$ netcat 192.168.178.29 2000 < user/demo_script2
```
The ESP will store the file and immediatly checks the syntax of the script:
```
CMD>script 2000
Waiting for script upload on port 2000
Script upload completed (451 Bytes)
Syntax okay
CMD>
```

You can examine the currently loaded script using the "show script" command. It only displays about 1KB of a script. If you need to see more, use "show script <line_no>" with a higher starting line. Newly loaded scripts are stored persistently in flash and will be executed after next reset if they contain no syntax errors. "script delete" stops script execution and deleted a script from flash.

# NTP Support
NTP time is supported and timestamps are only available if the sync with an NTP server is done. By default the NTP client is enabled and set to "1.pool.ntp.org". It can be changed by setting the config parameter "ntp_server" to a hostname or an IP address. An ntp_server of "none" will disable the NTP client. Also you can set the "ntp_timezone" to an offset from GMT in hours. The system time will be synced with the NTP server every "ntp_interval" seconds. Here it uses NOT the full NTP calculation and clock drift compensation. Instead it will just set the local time to the latest received time.

After NTP sync has been completed successfully once, the local time will be published every second under the topic "$SYS/broker/time" in the format "hh:mm:ss". You can also query the NTP time using the "time" command from the commandline. 

- set ntp_server _IP_or_hostname_: sets the name or IP of an NTP server (default "1.pool.ntp.org", "none" disables NTP)
- set ntp_interval _interval_: sets the NTP sync interval in seconds (default 300)
- set ntp_timezone _tz_: sets the timezone in hours offset (default 0)
- time: prints the current time as ddd hh:mm:ss

# mDNS
mDNS is supported and depending on "mdns_mode" the broker responds on the name "mqtt.local" with one of its two addresses:

- set mdns_mode [0|1|2]: selects, which interface address should be announced via mDNS (0=none (default), 1 = STA, 2 = SoftAP)

# Building and Flashing
The code can be used in any project that is compiled using the NONOS_SDK or the esp-open-sdk. Also the sample code in the user directory can be build using the standard SDKs after adapting the variables in the Makefile.

Build the esp_uMQTT_broker firmware with "make". "make flash" flashes it onto an esp8266.
 
If you want to use the precompiled binaries from the firmware directory you can flash them directly on an ESP8266, e.g. with

```bash
$ esptool.py --port /dev/ttyUSB0 write_flash -fs 32m 0x00000 firmware/0x00000.bin 0x10000 firmware/0x10000.bin

```
# The MQTT broker library
Thanks to Tuan PM for sharing his MQTT client library https://github.com/tuanpmt/esp_mqtt as a basis with us. The modified code still contains the complete client functionality from the original esp_mqtt lib, but it has been extended by the basic broker service.

The broker does support:
- MQTT protocoll versions v3.1 and v3.1.1 simultaniously
- a smaller number of clients (at least 8 have been tested, memory is the issue)
- retained messages
- LWT
- QoS level 0
- username/password authentication
 
The broker does not yet support:
- QoS levels other than 0
- many TCP(=MQTT) clients
- non-clear sessions
- TLS
"
# Using the esp_uMQTT_broker in an Arduino project
There is a fast-and-dirty hack to add the pure broker functionality (not the CLI and the scripting) to any ESP Arduino project:

- Go to the install directory of the ESP8266 support package (something like: "[yourArduinoDir]/hardware/esp8266com/esp8266")
- Look for the file "platform.txt"
- Search for the line with "compiler.c.elf.libs"
- Add "-lmqtt" to the libs. Now it should look like:
```
compiler.c.elf.libs=-lm -lgcc -lhal -lphy -lpp -lnet80211 -lwpa -lcrypto -lmain -lwps -laxtls -lsmartconfig -lmesh -lwpa2 -lmqtt {build.lwip_lib} -lstdc++
```
- From this directory go to "cd tools/sdk/lib".
- Copy "libmqtt.a" from the "firmware" directory of this repository into that location (where the other C-libs of the SDK are).
- From this directory go to "cd ../include".
- Copy "mqtt_server.h" from the "Arduino" directory of this repository into that location (where the other include files of the SDK are).
- Now you can use it in your sketch. Just include
```c
#include "mqtt_server.h"
```
Now you can use the API as described in the next subsection.

Sample: in the Arduino setup() initialize the WiFi connection (client or SoftAP, whatever you need) and somewhere at the end add these line:
```c
MQTT_server_start(1883, 30, 30);
```

The MQTT server will now run in the background and you can connect with any MQTT client. Your Arduino project might do other application logic in its loop. Let me know, if you need more available APIs to the broker from Adrduino code.

# Using the Source Code
The complete broker functionality is included in the mqtt directory and can be integrated into any NONOS SDK (or ESP Arduino) program ("make -f Makefile.orig lib" will build the mqtt code as a C library). You can find a minimal demo in the directory "user_basic". Rename it to "user", adapt "user_config.h", and do the "make" to build a small demo that just starts an MQTT broker without any additional logic.

The broker is started by simply including:

```c
#include "mqtt_server.h"
```
and then calling
```c
bool MQTT_server_start(uint16_t portno, uint16_t max_subscriptions, uint16_t max_retained_topics);
```
in the "user_init()" (or Arduino "setup()") function. Now it is ready for MQTT connections on all activated interfaces (STA and/or AP). Please note, that the lib uses two tasks (with prio 1 and 2) for client and broker. Thus, only task with prio 0 is left for a user application.

Your code can locally interact with the broker using these functions:

```c
bool MQTT_local_publish(uint8_t* topic, uint8_t* data, uint16_t data_length, uint8_t qos, uint8_t retain);
bool MQTT_local_subscribe(uint8_t* topic, uint8_t qos);
bool MQTT_local_unsubscribe(uint8_t* topic);

void MQTT_server_onData(MqttDataCallback dataCb);
```

With these functions you can publish and subscribe topics as a local client like you would with any remote MQTT broker. The provided dataCb is called on each reception of a matching topic, no matter whether it was published from a remote client or the "MQTT_local_publish()" function.

Username/password authentication is provided with the following interface:

```c
typedef bool (*MqttAuthCallback)(const char* username, const char *password, struct espconn *pesp_conn);

void MQTT_server_onAuth(MqttAuthCallback authCb);

typedef bool (*MqttConnectCallback)(struct espconn *pesp_conn);

void MQTT_server_onConnect(MqttConnectCallback connectCb);
```

If an *MqttAuthCallback* function is registered with MQTT_server_onAuth(), it is called on each connect request. Based on username, password, and optionally the connection info (e.g. the IP address) the function has to return *true* for authenticated or *false* for rejected. If a request provides no username and/or password these parameter strings are empty. If no *MqttAuthCallback* function is set, each request will be admitted.

The *MqttConnectCallback* function does a similar check for the connection, but it is called right after the connect request before any internal status is allocated. This is done in order to reject requests from unautorized clients in an early stage.
