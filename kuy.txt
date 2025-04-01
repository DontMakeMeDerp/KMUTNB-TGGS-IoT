#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BME680.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "DHT.h"
// Include the DHT11 library for interfacing with the sensor.
#include <DHT11.h>
#include <BH1750.h>
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>  // Include the Arduino JSON library

#define DHTPIN 2     // Pin connected to the data pin of the DHT sensor
#define DHTTYPE DHT11   // DHT11 or DHT22
// OLED DEFINE
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define OLED_RESET    -1
#define SCREEN_ADDRESS 0x3C  // I2C address of the display

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
WiFiClient espClient;
PubSubClient client(espClient);
DHT dht(DHTPIN, DHTTYPE);

const int relayFanPin = 26;  // Relay control pin
const int relayHumidPin = 27;  // Relay control pin
const int aqltPin = 34;

// Wi-Fi credentials
const char* ssid = "Exploratory2";
const char* password = "!tggs2025";

// MQTT Broker settings
const char* mqttServer = "202.44.44.233";
const int mqttPort = 1883;
const char* mqttTopic = "kerk/json";  // The topic to publish to

const int aqltStatus = LOW;

/* USE FUNCTION */
void setup() {
  Serial.begin(115200);
  MQTTSetup();
  pinMode(relayFanPin, OUTPUT);
  pinMode(relayHumidPin, OUTPUT);
  lightSetup();
  hNtSetup();
  aqltSetup();
  PMSetup();
  oledSetup();
}

void loop() {
  lightLoop();
  hNtLoop();
  aqltLoop();
  PMLoop();
  oledLoop();
  MQTTLoop();
  delay(1000);
}

/* Light Sensor SETUP */
BH1750 lightSensor;
void lightSetup() {
    Wire.begin();  // Initialize I2C (SDA=GPIO21, SCL=GPIO22)

    if (lightSensor.begin()) {
        Serial.println("BH1750 sensor initialized successfully.");
    } else {
        Serial.println("Error: Failed to initialize BH1750 sensor.");
        while (1);  // Stop execution if sensor fails
    }
}

void lightLoop() {
    float lux = lightSensor.readLightLevel();  // Read light level

    Serial.print("Light Level: ");
    Serial.print(lux);
    Serial.println(" lux");
}

/* Humidity and Temperature SETUP */
void hNtSetup() {
    // Uncomment the line below to set a custom delay between sensor readings (in milliseconds).
    // dht11.setDelay(500); // Default delay is 500ms.
}

void hNtLoop() {
    int temperature = 0;
    int humidity = 0;

    // Read temperature and humidity values from the DHT11 sensor.
    int result = dht11.readTemperatureHumidity(temperature, humidity);

    // Check if the reading is successful.
    if (result == 0) {
        Serial.print("Temperature: ");
        Serial.print(temperature);
        Serial.print(" °C\tHumidity: ");
        Serial.print(humidity);
        Serial.println(" %");
    } else {
        // Print an error message if the reading fails.
        Serial.println(DHT11::getErrorString(result));
    }
}

/* Air Quality SETUP */
void aqltSetup() {
  pinMode(aqltPin, INPUT);
  delay(2000);
}

void aqltLoop() {
  airStatus = digitalRead(aqltPin);
  if (aqltStatus == HIGH){
    Serial.println("Poor Air Quality");
  } else {
    Serial.println("Good Air Quality");
  }
}

/* OLED SETUP */
void oledSetup() {
    if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
        Serial.println(F("SSD1306 allocation failed"));
        for (;;);
    }
    display.clearDisplay();
}

void oledLoop() {
    // Display text
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 10);
    display.println("Hello, OLED!");
    display.display();
}

/* PM2.5 SETUP*/
SDS011 sds;
void PMSetup(){
  Serial2.begin(9600, SERIAL_8N1, 16, 17);
  sds.begin(&Serial2);
  Serial.println("SDS011 Sensor Test Started...")
}

void PMLoop(){
  float pn25, pm10;
  if (sds.read(&pm25, &pm10) == 0){
    Serial.printf("PM2.5: %.2f µg/m^3, PM10: %.2f µg/m^3 \n", pn25, pm10);
  } else {
    Serial.println("Failed to read SDS011 Sensor.")
  }
}

/* MQTT SETUP */
void MQTTSetup() {
  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to Wi-Fi");

  // Connect to MQTT broker
  client.setServer(mqttServer, mqttPort);
  while (!client.connected()) {
    String client_id = "esp32-client-";
    client_id += String(WiFi.macAddress());
    if (client.connect(client_id.c_str())) {
      Serial.println("Connected to MQTT broker");
    } else {
      Serial.print("Failed to connect, retrying...");
      delay(5000);
    }
  }
}

void MQTTLoop() {
  // Ensure the MQTT client remains connected
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Create a JSON object
  StaticJsonDocument<200> doc;
  // Add sensor data to the JSON object
  doc["temperature"] = temperature;  // Example: Temperature in Celsius
  doc["humidity"] = humidity;       // Example: Humidity in percentage
  doc["airquality"] = aqltStatus;
  doc["light"] = lux;
  doc["PM2.5"] = pm2_5;


  // Serialize the JSON object to a string
  char jsonBuffer[512];
  serializeJson(doc, jsonBuffer);

  // Publish the JSON string to MQTT topic
  if (client.publish(mqttTopic, jsonBuffer)) {
    Serial.println("Data published successfully");
  } else {
    Serial.println("Failed to publish data");
  }
}

// Function to reconnect to the MQTT broker
void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    String client_id = "esp32-client-";
    client_id += String(WiFi.macAddress());
    if (client.connect(client_id.c_str())) {
      Serial.println("Connected to MQTT broker");
    } else {
      Serial.print("Failed to connect. Retrying in 5 seconds...");
      delay(5000);
    }
  }
}