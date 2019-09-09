---


---

<h1 id="serial2mqtt">serial2mqtt</h1>
<p>For a complete view : <a href="https://vortex314.github.io/Serial2Mqtt.html">with UML sequence diagrams</a></p>
<p>MQTT for all micro-controllers ! The purpose is to offer MQTT publisher/subscriber functionality to all small micro controllers. Those with just a UART or USB interface.<br>
Example : a cheap STM32 board on ebay.<br>
<img src="https://img.staticbg.com/thumb/view/oaupload/banggood/images/8F/4A/3db92309-2e0b-4e4d-b2f1-9b017877ff42.jpg" alt="Afbeeldingsresultaat voor stm32 maple mini"><img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSrCMlaYevJNoWyIXyfdzTFL8nwC8WI-fKORO-c50cjDWjGZCqmZA" alt="Afbeeldingsresultaat voor lm4f120h5qr"><img src="http://www.rogerclark.net/wp-content/uploads/2014/11/STM32Mini-300x300.jpg" alt="Afbeeldingsresultaat voor ebay stm32"><br>
This program will act as a full MQTT Client gateway and make integration as simple as possible.<br>
This was created because Ethernet or WiFi is still absent in most ( cheap ) controllers .<br>
Also the concept behind is that a central PC or Raspberry PI can act as the intelligent mind behind commodity components.</p>
<p><img src="http://drive.google.com/uc?export=view&amp;id=1rGeHOaMEGLJJqxFsd5fnaAE7N1DHoJUI" alt="enter image description here"></p>
<p>Arduino Sample program to communicate with the serial2mqtt  gateway</p>
<pre><code>#include &lt;ArduinoJson.h&gt;

class Mqtt {
  public:
    static String device;
    static void publish( String topic, String message, int qos = 0, bool retained = false ) {
      StaticJsonBuffer&lt;200&gt; jsonBuffer;
      JsonObject&amp; data = jsonBuffer.createObject();
      data["cmd"] = "MQTT-PUB";
      data["topic"] = "src/" + device + "/" + topic;
      data["message"] = message;
      if ( qos != 0 ) data["qos"] = qos;
      if ( retained) data["retained"] = retained;
      data.printTo(Serial);
      Serial.println();
    }
    static void handleLine(String&amp; line) {
      StaticJsonBuffer&lt;200&gt; jsonBuffer;
      JsonObject&amp; root = jsonBuffer.parseObject(line);
      onMqttMessage(root["topic"], root["message"], root["qos"], root["retained"]);
    }

    static void onMqttMessage(String topic, String message, int qos, bool retained) {
    // add your own subscriber here 
      Serial.printf(" Mqtt Message arrived");
    }
};
// create a name for this device
String Mqtt::device = "ESP32"-" + String((uint32_t)ESP.getEfuseMac(), HEX);

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
  while (!Serial) {
    ; // wait for serial port to connect. Needed for native USB port only
  }
}

String line;
void loop() {
  while (Serial.available()) {
    char ch = Serial.read();
    if ( ch == '\r' || ch == '\n' ) {
      Mqtt::handleLine(line);
      line = "";
    } else
      line += ch;
  }
  digitalWrite(LED_BUILTIN, HIGH);   // turn the LED on (HIGH is the voltage level)
  delay(100);                       // wait for a 0.1 second
  digitalWrite(LED_BUILTIN, LOW);    // turn the LED off by making the voltage LOW
  delay(100);                       // wait for a 0.1 second
  Mqtt::publish( "system/upTime", String(millis(), 10), 0, false);
  Mqtt::publish("system/host", Mqtt::device, 0, false);
  Mqtt::publish("system/alive", "true", 0, false);
  }
</code></pre>
<h2 id="working-assumptions-and-features">Working assumptions and features</h2>
<ul>
<li>Topic Names<br>
–The design will take into account some assumptions about topic names and tree-structure to make it simple to use.<br>
Structure topic to and from  device :<br>
– dst/DEVICE/SERVICE/PROPERTY<br>
– src/DEVICE/SERVICE/PROPERTY<br>
– if DEVICE is not known yet the serial2mqtt will subscribe to the dst/HOST.PORT/serial2mqtt/# , where PORT is for example ttyUSB0</li>
<li>Serial messages will be  <strong>JSON</strong> array or object<br>
– JSON will be text delimited by newlines</li>
<li>Through the same communication, debugging logs can be handled without disturbing the mqtt flow. Any line that doesn’t start with ‘{’ or be a valid JSON is considered log.</li>
<li>the serial2mqtt establishes the client MQTT link and subscribes to dst/DEVICE/# when DEVICE is known.</li>
<li>when there is a big delay on the serial2mqtt serial input, it will do a serial disconnect and connect attempt , to unlock USB ports</li>
<li>serial2mqtt is event driven and as much as possible unblocking using MQTT in Async mode</li>
<li>one instance of serial2mqtt should be able to handle different serial ports</li>
<li>USB devices coming and going should be tracked by serial2mqtt</li>
<li>Configuration can be command line and config file driven ( JSON ). command line overrides config settings.</li>
</ul>
<h2 id="optional">Optional</h2>
<p>The serial2mqtt should be able to reset the device ( hard reset )</p>
<ul>
<li>The serial2mqtt should be able to program new code into the device</li>
<li>serial2mqtt should be able to program the device through the serial interface, for this purpose a third party app will be launched with the concerned serial port as argument.</li>
</ul>
<h1 id="protocol">Protocol</h1>
<h2 id="json-text">JSON TEXT</h2>
<h3 id="json-array">JSON ARRAY</h3>
<p>Example : [“0000”,1,“mytopic”,“3.141592653”]</p>
<pre><code>[&lt;CRC&gt;,&lt;COMMAND&gt;,&lt;TOPIC&gt;,&lt;MESSAGE&gt;,&lt;QOS&gt;,&lt;RETAIN&gt;] 
* QOS and retain are optional
&lt;CRC&gt; : can be checked or not, is calculated on the total JSON string based on the message containing "0000" as temporary CRC. When calculated is in HEX format.
* COMMAND 0:PING,1:PUBLISH,2:PUBACK,3:SUBSCRIBE,4:SUBACK,...
* ping : ["0000",0,"someText"]
* publish : ["ABCD",1,"dst/topic1","message1",0,0]
* subscribe : ["ABCD",3,"dst/myTopic"]
* QOS : 0,1,2 : for QOS, default 0
* RETAIN : 0 or 1 for true or false, default 0
</code></pre>
<h3 id="json-object">JSON OBJECT</h3>
<pre><code>    Example : { "cmd":"MQTT-PUB","topic":"src/device/service/property","message":"1234.66","qos":0,"retained":false }\n
</code></pre>
<h2 id="connection-setup">CONNECTION SETUP</h2>
<div class="mermaid"><svg xmlns="http://www.w3.org/2000/svg" id="mermaid-svg-d7oBhroRaf1A2HqF" height="100%" width="100%" style="max-width:650px;" viewBox="-50 -10 650 605"><g></g><g><line id="actor23" x1="75" y1="5" x2="75" y2="594" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="0" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">µC</tspan></text></g><g><line id="actor24" x1="275" y1="5" x2="275" y2="594" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="200" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">serial2mqtt</tspan></text></g><g><line id="actor25" x1="475" y1="5" x2="475" y2="594" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="400" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">MQTT Broker</tspan></text></g><defs><marker id="arrowhead" refX="5" refY="2" markerWidth="6" markerHeight="4" orient="auto"><path d="M 0,0 V 4 L6,2 Z"></path></marker></defs><defs><marker id="crosshead" markerWidth="15" markerHeight="8" orient="auto" refX="16" refY="4"><path fill="black" stroke="#000000" stroke-width="1px" d="M 9,2 V 6 L16,4 Z" style="stroke-dasharray: 0, 0;"></path><path fill="none" stroke="#000000" stroke-width="1px" d="M 0,1 L 6,7 M 6,1 L 0,7" style="stroke-dasharray: 0, 0;"></path></marker></defs><g><rect x="270" y="67" fill="#f4f4f4" stroke="#666" width="10" height="185" rx="0" ry="0"></rect></g><g><text x="172.5" y="93" class="messageText" style="text-anchor: middle;">CONNECT</text><line x1="270" y1="100" x2="75" y2="100" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="377.5" y="128" class="messageText" style="text-anchor: middle;">CONNECT(broker,port)</text><line x1="280" y1="135" x2="475" y2="135" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="377.5" y="163" class="messageText" style="text-anchor: middle;">CONNACK</text><line x1="475" y1="170" x2="280" y2="170" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><rect x="300" y="180" fill="#EDF2AE" stroke="#666" width="150" height="37" rx="0" ry="0" class="note"></rect><text x="296" y="204" fill="black" class="noteText"><tspan x="316" fill="black">connect serial</tspan></text></g><g><text x="172.5" y="245" class="messageText" style="text-anchor: middle;">{"topic":"src/DEVICE/SERVICE/PROP",...}</text><line x1="75" y1="252" x2="270" y2="252" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="270" y="254" fill="#f4f4f4" stroke="#666" width="10" height="68" rx="0" ry="0"></rect></g><g><text x="377.5" y="280" class="messageText" style="text-anchor: middle;">SUBSCRIBE("dst/DEVICE/</text><line x1="280" y1="287" x2="475" y2="287" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="377.5" y="315" class="messageText" style="text-anchor: middle;">PUBLISH("src/DEVICE/SERVICE/PROP",...)</text><line x1="280" y1="322" x2="475" y2="322" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="175" y="350" class="messageText" style="text-anchor: middle;">{"topic":"src/DEVICE/SERVICE/PROP1",...}</text><line x1="75" y1="357" x2="275" y2="357" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="270" y="359" fill="#f4f4f4" stroke="#666" width="10" height="33" rx="0" ry="0"></rect></g><g><text x="377.5" y="385" class="messageText" style="text-anchor: middle;">PUBLISH("src/DEVICE/SERVICE/PROP1",...)</text><line x1="280" y1="392" x2="475" y2="392" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><rect x="100" y="402" fill="#EDF2AE" stroke="#666" width="150" height="37" rx="0" ry="0" class="note"></rect><text x="96" y="426" fill="black" class="noteText"><tspan x="116" fill="black">no more messages after 5 sec, serial2mqtt disconnects serial port and tries to reconnect. MQTT connection always open.</tspan></text></g><g><text x="175" y="467" class="messageText" style="text-anchor: middle;">DISCONNECT</text><line x1="275" y1="474" x2="75" y2="474" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><text x="175" y="502" class="messageText" style="text-anchor: middle;">CONNECT</text><line x1="275" y1="509" x2="75" y2="509" class="messageLine1" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></g><g><rect x="0" y="529" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="561.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">µC</tspan></text></g><g><rect x="200" y="529" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="561.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">serial2mqtt</tspan></text></g><g><rect x="400" y="529" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="561.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">MQTT Broker</tspan></text></g></svg></div>
<h1 id="programming-through-serial2mqtt">Programming through serial2mqtt</h1>
<p>A command line utility will send a single mqtt request to the serial2mqtt gateway to program the microcontroller.</p>
<div class="mermaid"><svg xmlns="http://www.w3.org/2000/svg" id="mermaid-svg-XiTSH10aS4IihOCr" height="100%" width="100%" style="max-width:850px;" viewBox="-50 -10 850 476"><g></g><g><line id="actor26" x1="75" y1="5" x2="75" y2="465" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="0" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">µC</tspan></text></g><g><line id="actor27" x1="275" y1="5" x2="275" y2="465" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="200" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">serial2mqtt</tspan></text></g><g><line id="actor28" x1="475" y1="5" x2="475" y2="465" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="400" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">MQTT Broker</tspan></text></g><g><line id="actor29" x1="675" y1="5" x2="675" y2="465" class="actor-line" stroke-width="0.5px" stroke="#999"></line><rect x="600" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="32.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">programmer CLI</tspan></text></g><defs><marker id="arrowhead" refX="5" refY="2" markerWidth="6" markerHeight="4" orient="auto"><path d="M 0,0 V 4 L6,2 Z"></path></marker></defs><defs><marker id="crosshead" markerWidth="15" markerHeight="8" orient="auto" refX="16" refY="4"><path fill="black" stroke="#000000" stroke-width="1px" d="M 9,2 V 6 L16,4 Z" style="stroke-dasharray: 0, 0;"></path><path fill="none" stroke="#000000" stroke-width="1px" d="M 0,1 L 6,7 M 6,1 L 0,7" style="stroke-dasharray: 0, 0;"></path></marker></defs><g><text x="575" y="93" class="messageText" style="text-anchor: middle;">PUBLISH("dst/drive/serial2mqtt/flash",flash image binary)</text><line x1="675" y1="100" x2="475" y2="100" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#crosshead)" style="fill: none;"></line></g><g><text x="375" y="128" class="messageText" style="text-anchor: middle;">PUBLISH</text><line x1="475" y1="135" x2="275" y2="135" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="270" y="137" fill="#f4f4f4" stroke="#666" width="10" height="243" rx="0" ry="0"></rect></g><g><text x="172.5" y="163" class="messageText" style="text-anchor: middle;">program flash image</text><line x1="270" y1="170" x2="75" y2="170" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="377.5" y="198" class="messageText" style="text-anchor: middle;">PUBLISH(logs)</text><line x1="280" y1="205" x2="475" y2="205" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="575" y="233" class="messageText" style="text-anchor: middle;">logs</text><line x1="475" y1="240" x2="675" y2="240" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="172.5" y="268" class="messageText" style="text-anchor: middle;">startup logs</text><line x1="75" y1="275" x2="270" y2="275" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="377.5" y="303" class="messageText" style="text-anchor: middle;">logs</text><line x1="280" y1="310" x2="475" y2="310" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="575" y="338" class="messageText" style="text-anchor: middle;">logs</text><line x1="475" y1="345" x2="675" y2="345" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><text x="172.5" y="373" class="messageText" style="text-anchor: middle;">MQTT Pub</text><line x1="75" y1="380" x2="270" y2="380" class="messageLine0" stroke-width="2" stroke="black" marker-end="url(#arrowhead)" style="fill: none;"></line></g><g><rect x="0" y="400" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="75" y="432.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="75" dy="0">µC</tspan></text></g><g><rect x="200" y="400" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="275" y="432.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="275" dy="0">serial2mqtt</tspan></text></g><g><rect x="400" y="400" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="475" y="432.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="475" dy="0">MQTT Broker</tspan></text></g><g><rect x="600" y="400" fill="#eaeaea" stroke="#666" width="150" height="65" rx="3" ry="3" class="actor"></rect><text x="675" y="432.5" dominant-baseline="central" alignment-baseline="central" class="actor" style="text-anchor: middle;"><tspan x="675" dy="0">programmer CLI</tspan></text></g></svg></div>
<h1 id="logging-through-serial2mqtt">Logging through serial2mqtt</h1>
<p>Everything that serial2mqtt receives on the serial port is also send on a topic.</p>
<h1 id="build-instructions">Build instructions</h1>
<ul>
<li>use Codelite ( optional )</li>
<li>clone eclipse/paho.mqtt.c</li>
<li>clone bblanchon/ArduinoJson</li>
<li>clone vortex314/Common</li>
<li>install libssl-dev ( apt-get  install libssl-dev )</li>
<li>build static library in paho.mqtt.c by using <a href="http://makePaho.sh">makePaho.sh</a></li>
<li>build libCommon.a via “make -f <a href="http://Common.mk">Common.mk</a>”</li>
<li>build serial2mqtt via “make -f <a href="http://serial2mqtt.mk">serial2mqtt.mk</a>”</li>
</ul>
<p><strong>Or just deploy the pre-build versions</strong> from the Debug directory , 2 versions available : Linux 64bits Intel and Raspberry Pi ARM. The armv6l also runs on raspberry pi 3. Watch out for the arch command below.</p>
<pre><code>wget https://github.com/vortex314/serial2mqtt/raw/master/Debug/serial2mqtt.`arch`.zip
wget https://github.com/vortex314/serial2mqtt/raw/master/serial2mqtt.json
unzip serial2mqtt.`arch`.zip
mv Debug/serial2mqtt.`arch` serial2mqtt
</code></pre>
<h1 id="tested">Tested</h1>
<ul>
<li>ESP32 NodeMCU</li>
</ul>
<h1 id="still-to-do">Still to do</h1>
<ul>
<li>logging mechanism - DONE</li>
<li>disconnect serial and retry to avoid locking USB ports after timeouts - DONE</li>
<li>write binary image to file and send to microcontroller by activating configured external command , example esptool or stm32flash</li>
<li>implement binary ? why should I ?</li>
<li>command line tool to flash and monitor logs.<br>
– s2m -f file.bin -m <a href="http://test.mosquitto.org">test.mosquitto.org</a> -t pi1-USB0<br>
– s2m -f file.bin -m <a href="http://test.mosquitto.org">test.mosquitto.org</a> -t steer.USB0</li>
<li>Both lines have the same destination, logical and physical destination , if steer device is connected to pi1 host.</li>
<li>add other MQTT config params in config file : user, pswd, clientId</li>
<li>test with Maple Mini</li>
<li>add static topic through config : “src/DEVICE/serial2mqtt/board” “ESP32-Nodemcu” , which will be published every 5 seconds</li>
<li>add “MQTT-SUB” command to give micro-controller control over topic subscription.</li>
<li>add log level as parameter -l ( T,D,I,W,E ) for TRACE,DEBUG,INFO,WARN,ERROR level</li>
</ul>
<h1 id="code-design">Code design</h1>
<p>Per serial port there is a main thread and mqtt threads for callback<br>
The main thread waits for events and handle these primarily. 2 timers in this thread are checked for expiry ( not time critical ) : serial-watchdog and mqtt-connect.</p>
<p>To avoid concurrency issues , the callbacks of the mqtt threads are communicated back by writing an event code on a pipe.<br>
The main threads waits on events : timeout of 1 sec, data on serial file-descriptor or pipe file-descriptor.<br>
The mqtt event of received message is handled directly by writing the message on the serial port.</p>

