# Easy Raspberry Pi IoT Server

This is the written guide corresponding to our video tutorial showing you how to set up a Raspberry Pi IoT server using MQTT, Node-RED, InfluxDB and Grafana. These applications will be run inside docker containers so they are a perfect introduction if you haven’t tried containerising your applications before!

https://www.youtube.com/watch?v=_DO2wHI6JWQ

We are using a Raspberry Pi 4 loaded with Raspbian OS 64-bit lite running in headless mode connected via SSH. To start off with we make sure everything is updated:

sudo apt update
sudo apt upgrade
Then we download IoTStack using the following command and reboot once the download is complete:

curl -fsSL https://raw.githubusercontent.com/SensorsIot/IOTstack/master/install.sh | bash
sudo shutdown -r now
Once the Pi is back online we can SSH back in and open the menu. First navigate into the IoTStack folder then run the menu script:

cd IOTstack/
./menu.sh
In this menu navigate with the up and down arrows then space to select. Enter will build the docker-compose.yml file. Ensure you select the following components:

Grafana
InfluxDB
Mosquitto
Node-RED
Portainer-CE
You can then start all these containers with the “Start Stack” command. The first time you do this it might take some time as it downloads everything. Once complete you can check everything is running with the command:

docker-compose ps
To create an InfluxDB database we need to enter the influx docker container then add the database as follows:

docker exec -it influxdb influx
CREATE DATABASE sensor_data
quit
Now you can navigate to the web interfaces for the applications you need to configure as per the video. The Node-RED JSON interpreter we used was this:

{
    "temperature": msg.payload.t,
    "pressure": msg.payload.p,
    "humidity": msg.payload.h,
    "gas": msg.payload.g
}
To check that data is being written to the database you can use the following sequence of commands:

docker exec -it influxdb influx
USE sensor_data
show measurements
select * from sensor_data
quit
Source Code for the Sensor Node
This code is run on an Arduino Uno with a drop in replacement DIP package in the form of the Jolly module. A BME680 temperature, pressure, humidity and gas resistance sensor attached over the I2C interface.

```
#include <ArduinoJson.h>
#include <ArduinoJson.hpp>
#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BME680.h"

Adafruit_BME680 bme;

const char* ssid = "XXXX";
const char* password = "XXXX";
const char* mqtt_server = "XXX.XXX.X.XX";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE	(50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(100);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.init(AP_STA_MODE);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  while (!Serial);
  Serial.println(F("BME680 test"));

  if (!bme.begin()) {
    Serial.println("Could not find a valid BME680 sensor, check wiring!");
    while (1);
  }

  // Set up oversampling and filter initialization
  bme.setTemperatureOversampling(BME680_OS_8X);
  bme.setHumidityOversampling(BME680_OS_2X);
  bme.setPressureOversampling(BME680_OS_4X);
  bme.setIIRFilterSize(BME680_FILTER_SIZE_3);
  bme.setGasHeater(320, 150); // 320*C for 150 ms

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {

  if (!client.connected()) {
    reconnect();
  }
  client.loop();
  StaticJsonDocument<32> doc;
  char output[55];

  long now = millis();
  if (now - lastMsg > 5000) {
    lastMsg = now;
    float temp = bme.readTemperature();
    float pressure = bme.readPressure()/100.0;
    float humidity = bme.readHumidity();
    float gas = bme.readGas()/1000.0;
    doc["t"] = temp;
    doc["p"] = pressure;
    doc["h"] = humidity;
    doc["g"] = gas;
    Serial.println("Read");

    serializeJson(doc, output);
    Serial.println(output);
    client.publish("/home/sensors", output);
    Serial.println("Sent");
  }
    
}

```