<h1 id="toc_0">Dash IoT Guide (Arduino SAMD NINA)</h1>

**12 February 2025**

This guide demonstrates how to make BLE, TCP and MQTT connections to an Arduino SAMD based IoT device using the Arduino or PlatformIO IDE and the **DashIO** Adruino library. It also shows how to load user controls (widgets) into your mobile device to monitor and control an IoT device.

The boards intended for in this guide are the **Arduino NANO 33 IoT** and **Arduino MKR 1010**, but it may work for other SAMD boards.

The MQTT connection discussed in this guide is SSL/TLS enabled and we highly recommend that you only use SSL/TLS connections. The <strong>Dash</strong> MQTT server only accepts SSL connections.

<h2 id="toc_1">Getting Started</h2>

<p>For the big picture on <strong>Dash</strong>, take a look at our website: <a href="https://dashio.io">dashio.io</a></p>

<p>For the Dash arduino library: <a href="https://github.com/dashio-connect/arduino-dashioSAMD_NINA">github.com/dashio-connect/arduino-dashioSAMD_NINA</a></p>

<h2 id="toc_2">Requirements</h2>

<p>Grab a SAMD Arduino board, Arduino IDE and follow this guide.</p>

You will need to install the <strong>Dash</strong> app on your mobile phone or tablet. 

If you'd like to connect to your IoT devices through the **Dash** MQTT broker, setup an account on the <a href="https://dashio.io">dashio.io</a> website.

<h2 id="toc_3">Install</h2>

<h3 id="toc_4">Arduino IDE</h3>

You will need to add the **DashioSAMD_NINA** library into your project.  It is included in the Arduino IDE Library manager. Search the library manager for the library titled "DashioSAMD_NINA" and install. The Arduino IDE may also ask you to install the following libraries (dependencies). Please make sure they are all installed from the Arduino IDE Library Manager.

- **Dashio** is the core messageing library used to manage messages.
- **ArduinoBLE** required for BLE messaging.
- **ArduinoMqttClient** required for MQTT messaging.
- **WiFiNINA_Generic** by Khoi Hoang is required for WiFI and TCP connectivity.
- **arduino-timer** by Michael Contreras is used for the WiFi and MQTT connection.
- **FlashStorage** required storing WiFi and login credentials etc.

<h3 id="toc_5">PlatformIO</h3>

Using **Dash** with the **PlatformIO** IDE is easy.

1. Install the **Dash** library into your project. The easiest way to do this is to download the <a href="https://github.com/dashio-connect/arduino-dashio">**arduino-dashio**</a> library  into the ***lib*** directory of your project.
2. Install the libraries listed above using the PlatformIO **Libraries** manager.
3. Edit the ***platformio.ini*** file for you project by adding ```lib_ldf_mode = deep``` to enable all the required files to be found and also add ```monitor_speed = 115200``` to set the required serial monitor speed.

Your finished ***platformio.ini*** file should look something like this:

```
[env:nano_33_iot]
platform = atmelsam
board = nano_33_iot
framework = arduino
lib_deps = 
	contrem/arduino-timer@^2.3.1
	arduino-libraries/WiFiNINA@^1.8.13
	arduino-libraries/ArduinoMqttClient@^0.1.6
	arduino-libraries/ArduinoBLE@^1.3.2
	cmaglie/FlashStorage@^1.0.0
lib_ldf_mode = deep
monitor_speed = 115200
```

<h2 id="toc_6">Guide</h2>

<h3 id="toc_7">BLE Basics</h3>

<p>Lets start with a simple BLE connection.</p>

```
#include "DashioSAMD_NINA.h"

DashDevice dashDevice("SAMD_NINA_Type");
DashBLE    ble_con(&dashDevice, true);

void setup() {
    Serial.begin(115200);
    delay(1000);

    ble_con.begin();
    dashDevice.setup(ble_con.macAddress(), "Bob Name"); // unique deviceID, and device name
}

void loop() {
    ble_con.run();
}
```

<p>This is about the fewest number of lines of code necessary to get talking to the <strong>Dash</strong> app. There is a lot happening under the hood to make this work. After the <code>#include "DashioSAMD_NINA.h"</code> we create a device with the <em>device_type</em> as its only attribute. We also create a BLE connection, with the newly created device being an attribute to the connection. The second attribute of the BLE connection enables the received and transmitted messages to be printed to the serial monitor to make debugging easier.</p>

<p>In the <code>setup()</code> function we start the BLE connection <code>ble_conn.begin()</code>. For BLE connections we start the connection before <code>dashDevice.setup</code>. This is to make sure we get a sensible value for the mac address, which is then supplied to the device as a unique <em>device_ID</em>.</p>

<p>We call the device setup function <code>dashDevice.setup(ble_con.macAddress(), "Bob Name");</code> with two parameters that describe the device to the <strong>Dash</strong> app to allow it uniquely identify each device. </p>

<p>This device is discoverable by the <strong>Dash</strong> app. You can also discover your IoT device using a third party BLE scanner (e.g. BlueCap or "nRF Connect"). The BLE advertised name of your IoT device will be a concatentation of "DashIO_" and the <em>device_type</em>.</p>

<p>Setup the Arduino IDE serial monitor to 115200 baud and run the above code. Then run the <strong>Dash</strong> app on your mobile device and you will see connection "WHO" messages on the Arduino serial monitor as the Dash app detects and communicates with your IoT device.</p>

<p><img src="https://dashio.io/wp-content/uploads/2022/09/attention_44_yellow.png" width="20"> <strong>Troubleshooting:</strong>
Occasionally, the <strong>Dash</strong> app is unable to discover a BLE connection to the IoT device. If this occurs, try deleting the the IoT device from the Bluetooth Settings of your phone or tablet.</p>

<p>Lets add Dial control messages that are sent to the <strong>Dash</strong> app every second. To do this we create a new task to provide a 1 second time tick and then send a Dial value message from the loop every second.</p>

```
#include "DashioSAMD_NINA.h"
#include <arduino-timer.h>

DashDevice dashDevice("SAMD_NINA_Type");
DashBLE    ble_con(&dashDevice, true);

auto timer = timer_create_default();
bool oneSecond = false; // Set by hardware timer every second.

// Timer Interrupt
static bool onTimerCallback(void *argument) {
  oneSecond = true;
  return true; // to repeat the timer action - false to stop
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    ble_con.begin();
    dashDevice.setup(ble_con.macAddress(), "Bob Name"); // unique deviceID, and device name

    timer.every(1000, onTimerCallback); // 1000ms
}

void loop() {
    timer.tick();
    ble_con.run();

    if (oneSecond) { // Tasks to occur every second
        oneSecond = false;
        ble_con.sendMessage(dashDevice.getDialMessage("D01", int(random(0, 100))));
    }
}
```

<p>The line <code>dashDevice.getDialMessage("D01", int(random(0, 100)))</code> creates the message with two parameters. The first parameter is the <em>control_ID</em> which identifies the specific Dial control in the <strong>Dash</strong> app and the second parameter is simply the Dial value.</p>

<p>Once again, run the above code and the <strong>Dash</strong> app. This time, a new "DIAL" messages will be seen on the serial monitor every second.</p>

<p>The next step is to show the Dial values from the messages on a control on the <strong>Dash</strong> app.</p>

<p>In the <strong>Dash</strong> app, tap the <img src="https://dashio.io/wp-content/uploads/2021/07/iot_blue_44.png" width="20"> <strong>All Devices</strong> button, followed by the <img src="https://dashio.io/wp-content/uploads/2021/07/magnifying_glass_44_blue.png" width="20"> <strong>Find New Device</strong> button. Then select the <strong>BLE Scan</strong> option to show a list of new IoT devices that are BLE enabled. Your IoT device shpuld be shown in the list. Select your device and from the next menu select <strong>Create Device View</strong>. This will create an empty Device View for your new IoT deivce. You can now add controls to the Device View:</p>

<h3 id="toc_8">Adding Controls to Dash App</h3>

Once you have discovered your IoT device in the **Dash** app and have a **Device View** available for editing, you can add controls to the **Device View**:

<ul>
<li><strong>Start Editing</strong>: Tap the <img src="https://dashio.io/wp-content/uploads/2021/07/pencil_44_blue.png" width="20"> <strong>Edit</strong> button (it not already in editing mode).</li>
<li><strong>Add Dial Control</strong>: Tap the <img src="https://dashio.io/wp-content/uploads/2021/07/add_44_blue.png" width="20"> <strong>Add Control</strong> button and select the Dial control from the list.</li>
<li><strong>Edit Controls</strong>: Tap the Dial control to select it. The <img src="https://dashio.io/wp-content/uploads/2021/07/spanner_44_button.png" width="20"> <strong>Control Settings Menu</strong> button will appear in the middle of the Control. The Control can then be dragged and resized (pinch). Tapping the <img src="https://dashio.io/wp-content/uploads/2021/07/spanner_44_button.png" width="20"> button allows you to edit the Control settings where you can setup the style, colors and other characteristics of your Control. Make sure the <em>Control_ID</em> is set to the same value that is used in the Dial messages (in this case it should be set to "D01").</li>
<li><strong>Quit editing</strong>: Tap the <img src="https://dashio.io/wp-content/uploads/2021/07/pencil_quit_44.png" width="20"> <strong>Edit</strong> button again.</li>
</ul>

<p>The Dial on the <strong>Dash</strong> app will now show the random Dial values as they arrive.</p>

<p>The next piece of the puzzle to consider is how your IoT device can receive data from the <strong>Dash</strong> app. Lets add a Knob and connect it to the Dial. </p>

<p>In the <strong>Dash</strong> app you will need to add a <strong>Knob</strong> control onto your <strong>Device View</strong>, next to your <strong>Dial</strong> control. Edit the <strong>Knob</strong> to make sure the <strong>Control ID</strong> of the Knob matches what you have used in your Knob messages (in this case it should be "KB01"), then quit edit mode.</p>

Continuing with the BLE example, in the Arduino code we need to respond to messages coming in from a Knob control that we just added to the <strong>Dash</strong> app. To make the changes to your IoT device we add a callback, <code>processIncomingMessage</code>, into the BLE connection with the <code>setCallback</code> function.

```
#include "DashioSAMD_NINA.h"

DashDevice dashDevice("SAMD_NINA_Type");
DashBLE    ble_con(&dashDevice, true);

int dialValue = 0;

void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            ble_con.sendMessage(message);
        }
        break;
    }
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    ble_con.setCallback(&processIncomingMessage);
    ble_con.begin();
    dashDevice.setup(ble_con.macAddress(), "Bob Name"); // unique deviceID, and device name
}

void loop() {
    ble_con.run();
}
```

<p>We obtain the Knob value from the message data payload that we receive in the <code>processIncomingMessage</code> function. We then create a Dial message with the value from the Knob and send this back to the <strong>Dash</strong> app. Remember to remove the timer and the associated Dial <code>ble_con.sendMessage</code> from the <code>loop()</code> function.</p>

<p>When you adjust the Knob on the <strong>Dash</strong> app, a message with the Knob value is sent your IoT device, which returns the Knob value into the Dial control, which you will see on the <strong>Dash</strong> app.</p>

<p>Finally, we should respond to the STATUS message from the <strong>Dash</strong> app. STATUS messages allows the IoT device to send initial conditions for each control to the <strong>Dash</strong> app as soon as a connection becomes active. Once again, we do this from the <code>processIncomingMessage</code> function and our complete code looks like this:</p>

```
#include "DashioSAMD_NINA.h"

DashDevice dashDevice("SAMD_NINA_Type");
DashBLE    ble_con(&dashDevice, true);

int dialValue = 0;

void processStatus(ConnectionType connectionType) {
    String message((char *)0);
    message.reserve(1024);

    message = dashDevice.getKnobMessage("KB01", dialValue);
    message += dashDevice.getDialMessage("D01", dialValue);

    ble_con.sendMessage(message);
}

void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            ble_con.sendMessage(message);
        }
        break;
    }
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    ble_con.setCallback(&processIncomingMessage);
    ble_con.begin();
    dashDevice.setup(ble_con.macAddress(), "Bob Name"); // unique deviceID, and device name
}

void loop() {
    ble_con.run();
}
```

<h3 id="toc_9">TCP Basics</h3>

<p>Lets have a look at a TCP connection. For this we also need WiFi so it&#39;s very slightly more complicated than BLE.</p>

```
#include "DashioSAMD_NINA.h"

char WIFI_SSID[] = "yourWiFiSSID";
char WIFI_PASSWORD[] = "yourWiFiPassword";

DashDevice dashDevice("SAMD_NINA_Type");
DashTCP    tcp_con(&dashDevice, true);
DashWiFi   wifi;

void setup() {
    Serial.begin(115200);
    delay(1000);
    
    dashDevice.setup(wifi.macAddress(), "Bob Name"); // unique deviceID, and device name
    wifi.attachConnection(&tcp_con);
    wifi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
    wifi.run();
}
```

<p>Similar to BLE communications, we create a device and a TCP connection, with the device being an attribute to the connection. We also provide a port and the third attribute of the TCP connection enables the received and transmitted messages to be printed to the serial monitor to help with debugging.</p>

<p>In the <code>setup()</code> function we setup the device setup with dashDevice.setup(wifi.macAddress(), "Bob Name");. The two parameters describe the device to the <strong>Dash</strong> app to allow it uniquely identify each device. The wifi object supplies the <em>mac</em> to the device, to be used as a unique <em>device_ID</em>.</p>

<p>We then attach the TCP connection to the <code>wifi</code> with <code>wifi.attachConnection(&tcp_con);</code>. This enables to <code>wifi</code> object to control the TCP connection from here on in.</p>

<p>We start the WiFi wifi.begin(WIFI<em>SSID, WIFI</em>PASSWORD); with the SSID and password.</p>

<p>In the code above, make sure you replace <strong>yourWiFiSSID</strong> with your WiFi SSID and <strong>yourWiFiPassword</strong> with your WiFI password. Setup the Arduino IDE serial monitor with 115200 baud and run the code. </p>

This device is discoverable by the <strong>Dash</strong> app. You can also discover your IoT device using a third party Bonjour/Zeroconf discovery tool. The mDNS service will be "\_DashIO.\_tcp." and individual IoT devices on this service are identified by their mac address (<em>device_ID</em>).

<p>Int the <code>loop()</code> function, <code>wifi.run();</code> is used to regularly check that the WiFi is connected and manage the attached connection. </p>

<p>Message processing for a TCP connection is done exactly the samy way as for the BLE connection, by adding the <code>processIncomingMessage</code> callback into the TCP connection during <code>setup</code> function. Our finished code, with Dial and Knob added is:</p>

```
#include "DashioSAMD_NINA.h"

char WIFI_SSID[] = "yourWiFiSSID";
char WIFI_PASSWORD[] = "yourWiFiPassword";

DashDevice dashDevice("SAMD_NINA_Type");
DashTCP    tcp_con(&dashDevice, true);
DashWiFi   wifi;

int dialValue = 0;

void processStatus(ConnectionType connectionType) {
    String message((char *)0);
    message.reserve(1024);

    message = dashDevice.getKnobMessage("KB01", dialValue);
    message += dashDevice.getDialMessage("D01", dialValue);

    tcp_con.sendMessage(message);
}

void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            tcp_con.sendMessage(message);
        }
        break;
    }
}

void setup() {
    Serial.begin(115200);
    delay(1000);
    
    tcp_con.setCallback(&processIncomingMessage);
    dashDevice.setup(wifi.macAddress(), "Bob Name"); // unique deviceID, and device name
    wifi.attachConnection(&tcp_con);
    wifi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
    wifi.run();
}
```

<p>Run the <strong>Dash</strong> app once more. Tap the <img src="https://dashio.io/wp-content/uploads/2021/07/iot_blue_44.png" width="20"> <strong>All Devices</strong> button, followed by the <img src="https://dashio.io/wp-content/uploads/2021/07/magnifying_glass_44_blue.png" width="20"> <strong>Find New Device</strong> button. Then select <strong>Manually Connect</strong> and create your TCP connection.</p>

For the next step, you will need to setup your **Dash** app with Dial and Knob controls, as described in the section above **Adding Controls to Dash App**.

<p>When you adjust the Knob on the <strong>Dash</strong> app, a message with the Knob value is sent your IoT device, which returns the Knob value into the Dial control, which you will see on the <strong>Dash</strong> app.</p>

<h3 id="toc_10">MQTT Basics</h3>

<p>Lastly, lets look at a MQTT connection. We also need the WiFi for MQTT.</p>

```
#include "DashioSAMD_NINA.h"

// WiFi
char WIFI_SSID[] = "yourWiFiSSID";
char WIFI_PASSWORD[] = "yourWiFiPassword";

// MQTT
char MQTT_USER[] = "yourMQTTuserName";
char MQTT_PASSWORD[] = "yourMQTTpassword";

DashDevice dashDevice("DashIO_Type");
DashMQTT   mqtt_con(&dashDevice, true, true);
DashWiFi   wifi;

void setup() {
    Serial.begin(115200);

    dashDevice.setup(wifi.macAddress(), "Bob Name"); // unique deviceID, and device name
    mqtt_con.setup(MQTT_USER, MQTT_PASSWORD);
    wifi.attachConnection(&mqtt_con);
    wifi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
    wifi.run();
}
```

<p>Once again we create a device and this time a MQTT connection, with the device being an attribute to the connection. The second attribute of the MQTT connection enables a push notification to be sent to your mobile phone when the IoT device reboots (<strong>dash</strong> MQTT broker only). The final attribute enables the received and transmitted messages to be printed to the serial monitor for easy debugging.</p>

<p>In the <code>setup()</code> function we setup the device with two parameters. These parameters describe the device to the <strong>Dash</strong> app to allow it uniquely identify each device. The wifi object supplies the mac address to the device, to be used as a unique <em>device_ID</em>.</p>

<p>For MQTT connections, the device must be setup before we begin the wifi.</p>

<p>The MQTT connection setup requires the MQTT broker <em>username</em> and <em>password</em>, supplied in the <code>mqtt_con.setup(MQTT_USER, MQTT_PASSWORD);</code> method.</p>

<p>We then attach the MQTT connection to the <code>wifi</code> with <code>wifi.attachConnection(&mqtt_con);</code>. This enables to <code>wifi</code> object to control the MQTT connection from here on in.</p>

<p>Finally, we start the WiFi with <code>wifi.begin(WIFI_SSID, WIFI_PASSWORD);</code>, including parameters for the SSID and password.</p>

<p><code>wifi.run();</code> is used to regularly check that the WiFi is connected and to manage the attached connection. </p>

<p>This device is discoverable by the <strong>Dash</strong> app.</p>

<p>In the code above, make sure you replace <strong>yourWiFiSSID</strong> with your WiFi SSID, <strong>yourWiFiPassword</strong> with your WiFI password, <strong>yourMQTTuserName</strong> with you <strong>dash</strong> account username and <strong>yourMQTTpassword</strong> with your <strong>dash</strong> account password. Setup the Arduino IDE serial monitor with 115200 baud and run the code. You will see messages on the Arduino serial monitor, and after a few attempts, your IoT device should connect to your WiFi and then connect to the <strong>dash</strong> MQTT server.</p>

<p>Message processing for a MQTT connection is done exactly the samy way as for the BLE connection, by adding the <code>processIncomingMessage</code> callback into the MQTT connection during <code>setup</code> function. Our finished code, with Dial and Knob added is:</p>

```
#include "DashioSAMD_NINA.h"

// WiFi
char WIFI_SSID[] = "yourWiFiSSID";
char WIFI_PASSWORD[] = "yourWiFiPassword";

// MQTT
char MQTT_USER[] = "yourMQTTuserName";
char MQTT_PASSWORD[] = "yourMQTTpassword";

DashDevice dashDevice("DashIO_Type");
DashMQTT   mqtt_con(&dashDevice, true, true);
DashWiFi   wifi;

int dialValue = 0;

void processStatus(ConnectionType connectionType) {
    String message((char *)0);
    message.reserve(1024);

    message = dashDevice.getKnobMessage("KB01", dialValue);
    message += dashDevice.getDialMessage("D01", dialValue);

    mqtt_con.sendMessage(message);
}

void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            mqtt_con.sendMessage(message);
        }
        break;
    }
}

void setup() {
    Serial.begin(115200);

    dashDevice.setup(wifi.macAddress(), "Bob Name"); // unique deviceID, and device name
    mqtt_con.setup(MQTT_USER, MQTT_PASSWORD);
    mqtt_con.setCallback(&processIncomingMessage);
    wifi.attachConnection(&mqtt_con);
    wifi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
    wifi.run();
}
```

<p>Run the <strong>Dash</strong> app once more. Tap the <img src="https://dashio.io/wp-content/uploads/2021/07/iot_blue_44.png" width="20"> <strong>All Devices</strong> button, followed by the <img src="https://dashio.io/wp-content/uploads/2021/07/magnifying_glass_44_blue.png" width="20"> <strong>Find New Devices</strong> button. Then select the <strong>My Devices On Dash</strong> option to show a list of new IoT devices that have announced themselves to the <strong>dash</strong> server. Your device will be shown (named "Bob Name"). Select your IoT device to add the MQTT connection to your existing IoT device with the TCP connection.</p>

For the next step, you will need to setup your **Dash** app with Dial and Knob controls, as described in the section above **Adding Controls to Dash App**.

<p>When you adjust the Knob on the <strong>Dash</strong> app, a message with the Knob value is sent your IoT device, which returns the Knob value into the Dial control, which you will see on the <strong>Dash</strong> app.</p>

<p>For MQTT connections operating through the <strong>dash</strong> MQTT broker, a service is available for sending alarms (push notifications) to your mobile device. These are handy for sending messages or alarms to the user when they are not currently using the <strong>Dash</strong> app. To send an alarm, simply call the <code>sendMessage</code> method with two attrbutes; the first being any message String and the second having <strong>alarm_topic</strong> specified.</p>

<div><pre><code class="language-none">mqtt_con.sendMessage("message", alarm_topic);</code></pre></div>

<h3 id="toc_11">Combining BLE, TCP and MQTT</h3>

<p>The <strong>Arduino NANO 33 IoT</strong> and <strong>Arduino MKR 1010</strong> are not capable of operating with bith BLE and WiFi operating at the same time. However, it is easy to switch between BLE and TCP or MQTT.</p>

When combining connections, it is useful to create a generic ```sendMessage``` function to reply to an incoming message for a specific connection type. This prevents sending unnecessary reply messages to all connections:

```
void sendMessage(ConnectionType connectionType, const String& message) {
    if (connectionType == TCP_CONN) {
        tcp_con.sendMessage(message);
    } else if (connectionType == BLE_CONN) {
        ble_con.sendMessage(message);
    } else {
        mqtt_con.sendMessage(message);
    }
}
```

<p>This is just the beginning and there is a lot more we can do. Take a look at the examples in the library to see more details.</p>

<h2 id="toc_12">Layout Configuration</h2>

<p><strong>Layout configuration</strong> allows the IoT device to hold a copy of the complete layout of the device as it should appear on the <strong>Dash</strong> app. It includes all infomration for the device, controls (size, colour, style etc.), device views and connections.</p>

<p>When the <strong>Dash</strong> app discovers a new IoT device, it will download the Layout Configuration from the IoT device to display the device controls they way they have been designed by the developer. This is particularly useful developing commercial products and distributing your IoT devices to other users.</p>

<p>To include the Layout Configuration in your IoT device, simply follow these steps:</p>

<ul>
<li><strong>Design your layout</strong> in the <strong>Dash</strong> app and include all controls and connections that you need in your layout.</li>
<li><strong>Export the layout</strong>: Tap on the <strong>Device</strong> button, then tap the <strong>Export Layout</strong> button.</li>
<li><strong>Select the provisioning setup</strong> that you want (see below for provisioning details) and tap the <strong>Export</strong> button. The Layout Configuration will be emailed to you.</li>
<li><strong>Copy and paste</strong> the C64 configuration text from the email into your Arduino code, assigning it to a pointer to store the text in program memory. Your C64 configuration text will be different to that shown below.</li>
<li><strong>Add the Layout Config Revision</strong> integer (CONFIG_REV) as a third attribute to the DashDevice object.</li>
</ul>

```
const char configC64Str[] PROGMEM =
"lVNNb9swDP0rhQ47CUPiLM2WW2S7bhEndh3V3TAMgxursRZH8mQ5Hy3y30f5I83W7lD4QpOPpPge+Yx84qPx9x8YTecBAesZVRVP"
"0RhNh2RbrarLW5HE9zNvE4ZeFa8QRqU+5AwA8yCaTXxwLKXQSuY3jskivb7BMJEGIj8EImI5S0rAa1UxjDbJHo37vR5GRaKY0HWS"
"E9dJiqVxkleAHUFcc123mQr5AMGM8VWmo0Rzica9j9ZliwhlycEnAEmD0LTO5G7Gxcw0anoeOswpe/gJviFGa6hty1wq8wjGiouQ"
"izXUSHmSX8k8l7uybt8WMu4O/o2ZMGB3PNXZWWWM9q/6WdbIGg1geg7v7B2B7YXvRA3v1P1KG8u5aV2zSdga7vyusfzAa4xJ7DTG"
"wrdpi7f91nDi+L7WsNOIBJHjRgsjklY5cHIFWi34E8QGwPFK8RQGqjaiRGPLMuSBKo2nG1pUmxOk/7fYrW6mNJEqZaoj5z7jmnWR"
"9UqkXYAYeRv/f7FUJaKst2N5ALZMyy45WRp1us1wQI6LDxftgph8aiJE7t9o14VeVTccRCA0MDI4g74Q1f+MEYex58nGdL0r0LEW"
"a+Kfn8uXXqx+E3JNPe/GyTLuFtNf+9uzcyGTCP4KxZa8rPd18A+ZNZfvvw/Dwnvuo95tnp/49xRjAr11Jf2h+eAlkgv9Iu5p82Ez"
"Hpg6a2C7c+pGb97ge66kElyDHAidH4x9HbVnQr0ovG5MQum8s7z2ZuwruJRnlLItX7IF01UBpQRoh3f8kWO9LH4WUmmcJmWGZ7eU"
"YuK7mNphM5FT58Wc7dp9f1xFbAvm8fgH";

DashDevice    dashDevice(DEVICE_TYPE, configC64Str, CONFIG_REV);
```

Here is our Knob and Dial example, with TCP and MQTT connections and Layout Configuration added:

```
#include "DashioSAMD_NINA.h"

const char configC64Str[] PROGMEM =
"lVNNb9swDP0rhQ47CUPiLM2WW2S7bhEndh3V3TAMgxursRZH8mQ5Hy3y30f5I83W7lD4QpOPpPge+Yx84qPx9x8YTecBAesZVRVP"
"0RhNh2RbrarLW5HE9zNvE4ZeFa8QRqU+5AwA8yCaTXxwLKXQSuY3jskivb7BMJEGIj8EImI5S0rAa1UxjDbJHo37vR5GRaKY0HWS"
"E9dJiqVxkleAHUFcc123mQr5AMGM8VWmo0Rzica9j9ZliwhlycEnAEmD0LTO5G7Gxcw0anoeOswpe/gJviFGa6hty1wq8wjGiouQ"
"izXUSHmSX8k8l7uybt8WMu4O/o2ZMGB3PNXZWWWM9q/6WdbIGg1geg7v7B2B7YXvRA3v1P1KG8u5aV2zSdga7vyusfzAa4xJ7DTG"
"wrdpi7f91nDi+L7WsNOIBJHjRgsjklY5cHIFWi34E8QGwPFK8RQGqjaiRGPLMuSBKo2nG1pUmxOk/7fYrW6mNJEqZaoj5z7jmnWR"
"9UqkXYAYeRv/f7FUJaKst2N5ALZMyy45WRp1us1wQI6LDxftgph8aiJE7t9o14VeVTccRCA0MDI4g74Q1f+MEYex58nGdL0r0LEW"
"a+Kfn8uXXqx+E3JNPe/GyTLuFtNf+9uzcyGTCP4KxZa8rPd18A+ZNZfvvw/Dwnvuo95tnp/49xRjAr11Jf2h+eAlkgv9Iu5p82Ez"
"Hpg6a2C7c+pGb97ge66kElyDHAidH4x9HbVnQr0ovG5MQum8s7z2ZuwruJRnlLItX7IF01UBpQRoh3f8kWO9LH4WUmmcJmWGZ7eU"
"YuK7mNphM5FT58Wc7dp9f1xFbAvm8fgH";

char WIFI_SSID[] = "yourWiFiSSID";
char WIFI_PASSWORD[] = "yourWiFiPassword";
char MQTT_USER[] = "yourMQTTuserName";
char MQTT_PASSWORD[] = "yourMQTTpassword";

DashDevice dashDevice("MKR1010_Type", configC64Str, 1);
DashTCP    tcp_con(&dashDevice, true);
DashMQTT   mqtt_con(&dashDevice, true, true);
DashWiFi   wifi;

int dialValue = 0;

void sendMessage(ConnectionType connectionType, const String& message) {
    if (connectionType == TCP_CONN) {
        tcp_con.sendMessage(message);
    } else if (connectionType == MQTT_CONN) {
        mqtt_con.sendMessage(message);
    }
}

void processStatus(ConnectionType connectionType) {
    String message((char *)0);
    message.reserve(1024);

    message = dashDevice.getKnobMessage("KB01", dialValue);
    message += dashDevice.getDialMessage("D01", dialValue);

    sendMessage(connectionType, message);
}

void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            sendMessage(messageData->connectionType, message);
        }
        break;
    }
}

void setup() {
    Serial.begin(115200);
    delay(1000);
    
    dashDevice.setup(wifi.macAddress(), "Bob Name"); // unique deviceID, and device name

    tcp_con.setCallback(&processIncomingMessage);
    wifi.attachConnection(&tcp_con);
    
    mqtt_con.setup(MQTT_USER, MQTT_PASSWORD);
    mqtt_con.setCallback(&processIncomingMessage);
    wifi.attachConnection(&mqtt_con);

    wifi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
    wifi.run();
}
```

<h2 id="toc_13">Provisioning</h2>

<p>Provisioning is the method by which the Dash app can send user credentials and other setup information to the IoT device and the IoT device will store this data to non-volatile memory.</p>

<p>Details of the provisioning functionality available in an IoT device are contained in the Layout Configuration and must be setup when exporting the Layout Configuration from the <strong>Dash</strong> app.</p>

<p>You will need to install the Arduino Flash Storage library. Search the arduino library manager for the library titled "Flash Storage" and install.</p>

<p>A provisioning library is available within the Dash library for the Arduino SAMD boards (DashProvisionSAMD.h and DashioProvisionSAMD.cpp) which stores the following provisioning data to Flash memory:</p>

<ul>
<li>Device Name</li>
<li>WiFi SSID</li>
<li>WiFI Password</li>
<li>Dash MQTT broker User Name</li>
<li>Dash MQTT broker Password</li>
</ul>

<p><img src="https://dashio.io/wp-content/uploads/2022/09/attention_44_yellow.png" width="20"> <strong>WARNING</strong>: Flash memory has a limited amount of write cycles. Typical, flash memory can perform about 10000 writes cycles to the same flash block before starting to "wear out" and begin to lose the ability to retain data. Also, when you reprogram the microcontroller, the provisioning data will be erased.</p>

<p>The provisioning library exposes the following structure to manade the provisioning data:</p>

<div><pre><code class="language-none">struct DeviceData {
    char deviceName[32];
    char wifiSSID[32];
    char wifiPassword[63];
    char dashUserName[32];
    char dashPassword[32];
    char saved;
};</code></pre></div>

<p>Use the provisioning library as follows:</p>

<p><strong>Include the library</strong></p>

<div><pre><code class="language-none">#include "DashProvisionSAMD.h"</code></pre></div>

<p><strong>Create a provisioning object</strong></p>

<div><pre><code class="language-none">DashProvision dashProvision(&dashDevice);</code></pre></div>

<p><strong>Setup provisioning</strong> in the <code>setup()</code> function. For the TCP connection, we assign the provisioning information to the WiFi object  or, for an MQTT connection, we assign the provisioning information to the WiFI object and MQTT connection object:</p>

```
// setup
DeviceData defaultDeviceData = {DEVICE_NAME, WIFI_SSID, WIFI_PASSWORD, MQTT_USER, MQTT_PASSWORD};
dashProvision.load(&defaultDeviceData, &onProvisionCallback);

dashDevice.setup(wifi.macAddress());
    
mqtt_con.setup(dashProvision.dashUserName, dashProvision.dashPassword);
mqtt_con.setCallback(&processIncomingMessage);
    
wifi.attachConnection(&mqtt_con);
wifi.begin(dashProvision.wifiSSID, dashProvision.wifiPassword);
```

<p>Here we update the provisioning object with the default provisioning data (using the <code>DeviceData</code> structure), then load the provisioning data. If the provisioning data has never been saved, then the the default provisionig data will be used. We also include a provision callback (discussed later).</p>

<p><strong>Add provisioning process message</strong> in the <code>processIncomingMessage</code> callback from the example above becomes:</p>

<div><pre><code class="language-none">void processIncomingMessage(MessageData *messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            sendMessage(messageData->connectionType, message);
        }
        break;
    default:
        dashProvision.processMessage(messageData);
        break;
    }
}</code></pre></div>

<p>Your IoT device will let the <strong>Dash</strong> app know what provisioning services are available when it down loads the <strong>layout configuration</strong>.</p>

<p>And finally, here is the <code>onProvisionCallback</code>:</p>

```
void onProvisionCallback(ConnectionType connectionType, const String& message, bool commsChanged) {
    sendMessage(connectionType, message);
    
    if (!commsChanged) { // Must be a name change
        mqtt_con.sendWhoAnnounce(); // Update to announce topic with new name
    }

    // Restart WiFi and MQTT
}
```

<p>The callback has two purposes. Firstly we send a message back to the Dash app to let it know the provisioning is complete. And secondly, if the WiFi or MQTT connection credentials have changed, we need to restart the WiFi or MQTT connection or reboot the processor for a fresh start.</p>

<p>To provision your device, run the <strong>Dash</strong> app and tap the <img src="https://dashio.io/wp-content/uploads/2021/07/iot_blue_44.png" width="20"> <strong>All Devices</strong> button and select your device. Tap the <img src="https://dashio.io/wp-content/uploads/2022/09/key_44_blue.png" width="20"> <strong>Provisioning</strong> button and follow the instructions.</p>

The final provisioning feature is the ability to "Reset" your IoT device. The **Dash** app provisioning menu also includes a "Reset Device" button. In your IoT device, you will receive an incoming message of ControlType = ***resetDevice*** which you can manage in your ```processIncomingMessage``` callback as shown in the following example:

```
void processIncomingMessage(MessageData * messageData) {
    switch (messageData->control) {
    case status:
        processStatus(messageData->connectionType);
        break;
    case knob:
        if (messageData->idStr == "KB01") {
            dialValue = messageData->payloadStr.toFloat();
            String message = dashDevice.getDialMessage("D01", dialValue);
            sendMessage(messageData->connectionType, message);
        }
        break;
    case resetDevice:
        sendMessage(messageData->connectionType, dashDevice.getResetDeviceMessage());
        delay(2000); // Make sure the above message is sent before resetting
        NVIC_SystemReset();
        break;
    default:
        dashProvision.processMessage(messageData);
        break;
    }
}
```

<h1 id="toc_14">Jump In and Build Your Own IoT Device</h1>

When you are ready to create your own IoT device, the Dash Arduino C++ Library will provide you with more details about what you need to know:

<a href="https://dashio.io/arduino-library/">https://dashio.io/arduino-library/</a>
