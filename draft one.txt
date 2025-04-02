#include <Wire.h>
#include <BH1750.h>
#include <DHT11.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <SDS011.h>

// Recommended values
#define TEMP_THRESHOLD 25   // Temperature threshold (°C)
#define HUMIDITY_THRESHOLD 40 // Humidity threshold (%)
#define LIGHT_LOW 10         // Low light threshold (lux)
#define LIGHT_HIGH 500       // High light threshold (lux)
#define PM25_LIMIT 35        // PM2.5 threshold (µg/m³)
#define PM10_LIMIT 50        // PM10 threshold (µg/m³)

// Pin Definitions
#define FAN_RELAY 26
#define HUMIDIFIER_RELAY 27
#define DHT_PIN 2

// Sensor Instances
BH1750 lightSensor;
DHT11 dht11(DHT_PIN);
Adafruit_SSD1306 display(128, 32, &Wire, -1);
SDS011 sds;

void sensorSetup() {
    Serial.begin(115200);
    Wire.begin();
    
    // BH1750 Setup
    if (lightSensor.begin()) {
        Serial.println("BH1750 initialized.");
    } else {
        Serial.println("BH1750 initialization failed!");
    }
    
    // OLED Setup
    if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
        Serial.println("SSD1306 initialization failed!");
        for (;;);
    }
    display.clearDisplay();
    display.display();
    
    // SDS011 Setup
    Serial2.begin(9600, SERIAL_8N1, 16, 17);
    delay(1000);
    sds.begin(&Serial2);
    Serial.println("SDS011 Sensor Ready");
    
    // Relay Setup
    pinMode(FAN_RELAY, OUTPUT);
    pinMode(HUMIDIFIER_RELAY, OUTPUT);
}

void readSensors() {
    // Read Light Level
    float lux = lightSensor.readLightLevel();
    Serial.printf("Light: %.2f lux\n", lux);
    
    // Read Temperature & Humidity
    int temp = 0, hum = 0;
    int dhtStatus = dht11.readTemperatureHumidity(temp, hum);
    if (dhtStatus == 0) {
        Serial.printf("Temperature: %d °C, Humidity: %d %%\n", temp, hum);
    } else {
        Serial.println("DHT11 read error.");
    }
    
    // Read PM2.5 & PM10
    float pm25, pm10;
    if (sds.read(&pm25, &pm10) == 0) {
        Serial.printf("PM2.5: %.2f µg/m³, PM10: %.2f µg/m³\n", pm25, pm10);
    } else {
        Serial.println("SDS011 read error.");
    }
    
    controlDevices(temp, hum, lux, pm25, pm10);
    updateOLED(lux, pm25, pm10);
}

void controlDevices(int temp, int hum, float lux, float pm25, float pm10) {
    // Control Fan based on Temperature
    if (temp > TEMP_THRESHOLD) {
        digitalWrite(FAN_RELAY, HIGH);
        Serial.println("Fan ON");
    } else {
        digitalWrite(FAN_RELAY, LOW);
    }
    
    // Control Humidifier based on Humidity
    if (hum < HUMIDITY_THRESHOLD) {
        digitalWrite(HUMIDIFIER_RELAY, HIGH);
        Serial.println("Humidifier ON");
    } else {
        digitalWrite(HUMIDIFIER_RELAY, LOW);
    }
}

void updateOLED(float lux, float pm25, float pm10) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    
    if (lux < LIGHT_LOW) {
        display.println("Too Dark!");
    } else if (lux > LIGHT_HIGH) {
        display.println("Too Bright!");
    }
    
    if (pm25 > PM25_LIMIT || pm10 > PM10_LIMIT) {
        display.printf("Air Quality Bad!\nPM2.5: %.2f\nPM10: %.2f\n", pm25, pm10);
    }
    
    display.display();
}

void setup() {
    sensorSetup();
}

void loop() {
    readSensors();
    delay(2000);
}
