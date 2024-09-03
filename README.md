# source-code


#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <Wire.h>
#include <Adafruit_SSD1306.h>
#include <time.h>
#include "secrets.h"
#include "DHT.h"

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define SCREEN_I2C_ADDR 0x3C  // Update if necessary

#define TIME_ZONE 5.5
#define BUTTON_PIN D7
#define BUZZER_PIN D6
#define LDR_PIN D5
#define DHTPIN D4
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

float h, t;
unsigned long lastMillis = 0;
unsigned long buttonPressTime = 0;
unsigned long debounceDelay = 50; // Debounce delay in milliseconds
bool isButtonPressed = false;
bool wasButtonPressed = false;
bool showTemperature = false;
bool showMessage = false;
bool alarmActive = false; // Track alarm status
String awsMessage = "";

#define AWS_IOT_PUBLISH_TOPIC   "esp8266/pub"
#define AWS_IOT_SUBSCRIBE_TOPIC "esp8266/sub"

WiFiClientSecure net;
BearSSL::X509List cert(cacert);
BearSSL::X509List client_crt(client_cert);
BearSSL::PrivateKey key(privkey);
PubSubClient client(net);

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Define 4 frames of the walking animation (replace these with actual frame data)
const byte PROGMEM frames[][512] = {
  {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,63,240,0,0,15,248,0,0,127,248,0,0,31,254,0,1,248,120,0,0,30,31,128,3,192,240,0,0,15,3,192,7,129,224,0,0,7,129,224,7,3,192,63,252,3,192,224,14,7,131,255,255,193,224,112,12,14,15,248,31,240,240,48,28,28,63,128,1,252,56,56,24,56,124,0,0,62,28,24,56,113,240,0,0,15,142,24,56,227,192,0,0,3,199,24,57,199,128,0,0,1,227,152,27,143,0,0,0,0,241,216,31,30,0,0,0,0,120,248,30,28,0,0,0,0,56,120,28,56,0,0,0,0,28,56,8,112,0,0,0,0,14,16,0,112,56,0,0,0,14,0,0,224,60,0,0,0,7,0,0,192,30,0,0,0,3,0,1,192,15,0,0,0,3,128,1,128,7,128,0,0,1,128,3,128,3,192,0,0,1,192,3,128,1,224,0,0,1,192,3,128,0,240,0,0,1,192,3,0,0,120,0,0,0,192,3,0,0,60,0,0,0,192,7,0,0,30,0,0,0,224,7,0,0,15,0,0,0,224,7,0,7,255,128,0,0,224,7,0,15,255,192,0,0,224,7,0,7,255,128,0,0,224,7,0,0,0,0,0,0,224,7,0,0,0,0,0,0,224,3,0,0,0,0,0,0,192,3,0,0,0,0,0,0,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,1,128,0,0,0,0,1,128,1,192,0,0,0,0,3,128,1,192,0,0,0,0,3,128,0,224,0,0,0,0,7,0,0,96,0,0,0,0,6,0,0,112,0,0,0,0,14,0,0,56,0,0,0,0,28,0,0,60,0,0,0,0,60,0,0,30,0,0,0,0,120,0,0,15,0,0,0,0,240,0,0,15,128,0,0,1,240,0,0,15,192,0,0,3,240,0,0,29,240,0,0,15,184,0,0,56,124,0,0,62,28,0,0,112,63,0,0,252,14,0,0,224,15,240,15,240,7,0,1,192,3,255,255,192,3,128,3,128,0,127,254,0,1,192,1,0,0,0,0,0,0,128,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
  {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,63,240,0,0,15,248,0,0,127,248,0,0,31,254,0,1,248,120,0,0,30,31,128,3,192,240,0,0,15,3,192,7,129,224,0,0,7,129,224,7,3,192,63,252,3,192,224,14,7,131,255,255,193,224,112,12,14,15,248,31,240,240,48,28,28,63,128,1,252,56,56,24,56,124,0,0,62,28,24,56,113,240,0,0,15,142,24,56,227,192,0,0,3,199,24,57,199,128,0,0,1,227,152,27,143,0,0,0,0,241,216,31,30,0,0,0,0,120,248,30,28,0,0,0,0,56,120,28,56,8,0,0,0,28,56,8,112,28,0,0,0,14,16,0,112,14,0,0,0,14,0,0,224,15,0,0,0,7,0,0,192,7,128,0,0,3,0,1,192,3,128,0,0,3,128,1,128,1,192,0,0,1,128,3,128,0,224,0,0,1,192,3,128,0,112,0,0,1,192,3,128,0,120,0,0,1,192,3,0,0,56,0,0,0,192,3,0,0,28,0,0,0,192,7,0,0,14,0,0,0,224,7,0,0,7,0,0,0,224,7,0,7,255,128,0,0,224,7,0,15,255,192,0,0,224,7,0,7,255,128,0,0,224,7,0,0,0,0,0,0,224,7,0,0,0,0,0,0,224,3,0,0,0,0,0,0,192,3,0,0,0,0,0,0,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,1,128,0,0,0,0,1,128,1,192,0,0,0,0,3,128,1,192,0,0,0,0,3,128,0,224,0,0,0,0,7,0,0,96,0,0,0,0,6,0,0,112,0,0,0,0,14,0,0,56,0,0,0,0,28,0,0,60,0,0,0,0,60,0,0,30,0,0,0,0,120,0,0,15,0,0,0,0,240,0,0,15,128,0,0,1,240,0,0,15,192,0,0,3,240,0,0,29,240,0,0,15,184,0,0,56,124,0,0,62,28,0,0,112,63,0,0,252,14,0,0,224,15,240,15,240,7,0,1,192,3,255,255,192,3,128,3,128,0,127,254,0,1,192,1,0,0,0,0,0,0,128,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
  {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,63,240,0,0,15,248,0,0,127,248,0,0,31,254,0,1,248,120,0,0,30,31,128,3,192,240,0,0,15,3,192,7,129,224,0,0,7,129,224,7,3,192,63,252,3,192,224,14,7,131,255,255,193,224,112,12,14,15,248,31,240,240,48,28,28,63,128,1,252,56,56,24,56,124,0,0,62,28,24,56,113,240,0,0,15,142,24,56,227,192,0,0,3,199,24,57,199,128,0,0,1,227,152,27,143,0,0,0,0,241,216,31,30,0,0,0,0,120,248,30,28,6,0,0,0,56,120,28,56,7,0,0,0,28,56,8,112,7,0,0,0,14,16,0,112,3,128,0,0,14,0,0,224,3,192,0,0,7,0,0,192,1,192,0,0,3,0,1,192,0,224,0,0,3,128,1,128,0,240,0,0,1,128,3,128,0,112,0,0,1,192,3,128,0,56,0,0,1,192,3,128,0,60,0,0,1,192,3,0,0,28,0,0,0,192,3,0,0,14,0,0,0,192,7,0,0,15,0,0,0,224,7,0,0,7,0,0,0,224,7,0,7,255,128,0,0,224,7,0,15,255,192,0,0,224,7,0,7,255,128,0,0,224,7,0,0,0,0,0,0,224,7,0,0,0,0,0,0,224,3,0,0,0,0,0,0,192,3,0,0,0,0,0,0,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,1,128,0,0,0,0,1,128,1,192,0,0,0,0,3,128,1,192,0,0,0,0,3,128,0,224,0,0,0,0,7,0,0,96,0,0,0,0,6,0,0,112,0,0,0,0,14,0,0,56,0,0,0,0,28,0,0,60,0,0,0,0,60,0,0,30,0,0,0,0,120,0,0,15,0,0,0,0,240,0,0,15,128,0,0,1,240,0,0,15,192,0,0,3,240,0,0,29,240,0,0,15,184,0,0,56,124,0,0,62,28,0,0,112,63,0,0,252,14,0,0,224,15,240,15,240,7,0,1,192,3,255,255,192,3,128,3,128,0,127,254,0,1,192,1,0,0,0,0,0,0,128,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
  {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,63,240,0,0,15,248,0,0,127,248,0,0,31,254,0,1,248,120,0,0,30,31,128,3,192,240,0,0,15,3,192,7,129,224,0,0,7,129,224,7,3,192,63,252,3,192,224,14,7,131,255,255,193,224,112,12,14,15,248,31,240,240,48,28,28,63,128,1,252,56,56,24,56,124,0,0,62,28,24,56,113,240,0,0,15,142,24,56,227,192,0,0,3,199,24,57,199,128,0,0,1,227,152,27,143,0,0,0,0,241,216,31,30,1,0,0,0,120,248,30,28,3,128,0,0,56,120,28,56,1,192,0,0,28,56,8,112,1,192,0,0,14,16,0,112,0,224,0,0,14,0,0,224,0,224,0,0,7,0,0,192,0,112,0,0,3,0,1,192,0,112,0,0,3,128,1,128,0,56,0,0,1,128,3,128,0,56,0,0,1,192,3,128,0,28,0,0,1,192,3,128,0,28,0,0,1,192,3,0,0,14,0,0,0,192,3,0,0,14,0,0,0,192,7,0,0,7,0,0,0,224,7,0,0,7,0,0,0,224,7,0,7,255,128,0,0,224,7,0,15,255,192,0,0,224,7,0,7,255,128,0,0,224,7,0,0,0,0,0,0,224,7,0,0,0,0,0,0,224,3,0,0,0,0,0,0,192,3,0,0,0,0,0,0,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,3,128,0,0,0,0,1,192,1,128,0,0,0,0,1,128,1,192,0,0,0,0,3,128,1,192,0,0,0,0,3,128,0,224,0,0,0,0,7,0,0,96,0,0,0,0,6,0,0,112,0,0,0,0,14,0,0,56,0,0,0,0,28,0,0,60,0,0,0,0,60,0,0,30,0,0,0,0,120,0,0,15,0,0,0,0,240,0,0,15,128,0,0,1,240,0,0,15,192,0,0,3,240,0,0,29,240,0,0,15,184,0,0,56,124,0,0,62,28,0,0,112,63,0,0,252,14,0,0,224,15,240,15,240,7,0,1,192,3,255,255,192,3,128,3,128,0,127,254,0,1,192,1,0,0,0,0,0,0,128,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}
};

#define FRAME_DELAY (50)  // Adjust this value for faster animation
#define FRAME_WIDTH (64)
#define FRAME_HEIGHT (64)
#define FRAME_COUNT (sizeof(frames) / sizeof(frames[0]))

void NTPConnect(void) {
  Serial.print("Setting time using SNTP");
  configTime(TIME_ZONE * 3600, 0, "pool.ntp.org", "time.nist.gov");
  time_t now = time(nullptr);
  time_t nowish = 1510592825;
  while (now < nowish) {
    delay(500);
    Serial.print(".");
    now = time(nullptr);
  }
  Serial.println("done!");
}

void messageReceived(char *topic, byte *payload, unsigned int length) {
  Serial.print("Received [");
  Serial.print(topic);
  Serial.print("]: ");
  awsMessage = "";
  for (int i = 0; i < length; i++) {
    awsMessage += (char)payload[i];
  }
  Serial.println(awsMessage);
  showMessage = true; // Set flag to show message on display
}

void connectAWS() {
  delay(3000);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  Serial.println(String("Attempting to connect to SSID: ") + String(WIFI_SSID));

  while (WiFi.status() != WL_CONNECTED) {
    displayLoadingScreen(); // Show loading animation
    delay(1000); // Adjust delay for animation speed
  }

  NTPConnect();

  net.setTrustAnchors(&cert);
  net.setClientRSACert(&client_crt, &key);

  client.setServer(MQTT_HOST, 8883);
  client.setCallback(messageReceived);

  Serial.println("Connecting to AWS IOT");

  while (!client.connect(THINGNAME)) {
    displayLoadingScreen(); // Show loading animation
    delay(1000); // Adjust delay for animation speed
  }

  if (!client.connected()) {
    Serial.println("AWS IoT Timeout!");
    return;
  }

  client.subscribe(AWS_IOT_SUBSCRIBE_TOPIC);
  Serial.println("AWS IoT Connected!");
}

void publishMessage() {
  StaticJsonDocument<200> doc;
  doc["time"] = millis();
  doc["humidity"] = h;
  doc["temperature"] = t;
  char jsonBuffer[512];
  serializeJson(doc, jsonBuffer);

  client.publish(AWS_IOT_PUBLISH_TOPIC, jsonBuffer);
}

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LDR_PIN, INPUT);

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_I2C_ADDR)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }

  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Connecting...");
  display.display();

  dht.begin();
  connectAWS();
}

void loop() {
  if (!client.connected()) {
    connectAWS();
  } else {
    client.loop();
  }

  int buttonState = digitalRead(BUTTON_PIN);
  int ldrState = digitalRead(LDR_PIN); // Read the LDR sensor

  if (buttonState == LOW) { // Button is pressed
    if (!isButtonPressed) {
      isButtonPressed = true;
      buttonPressTime = millis();
    } else if (millis() - buttonPressTime >= 2000 && !wasButtonPressed) { // Long press detected
      showTemperature = !showTemperature;
      showMessage = false; // Reset message display
      wasButtonPressed = true; // Prevent toggling until button is released
      if (showTemperature) {
        // Read and display the temperature and humidity
        updateTemperatureHumidity();
      } else {
        displayTime();
      }
    }
  } else {
    isButtonPressed = false;
    wasButtonPressed = false; // Reset after button release
  }

  if (showMessage) {
    displayMessage();
  } else if (!showTemperature) {
    displayTime();
  }

  // Adaptive brightness adjustment
  if (ldrState == HIGH) {
    // Bright environment (Light falling on LDR)
    display.ssd1306_command(SSD1306_SETCONTRAST);  // Command to adjust contrast
    display.ssd1306_command(0x10);  // Lower contrast (16)
  } else {
    // Dark environment (LDR covered)
    display.ssd1306_command(SSD1306_SETCONTRAST);  // Command to adjust contrast
    display.ssd1306_command(0xFF);  // Maximum contrast (255)
  }

  // Update temperature and humidity every 3 seconds
  if (millis() - lastMillis >= 3000) {
    lastMillis = millis();

    // Update temperature and humidity values
    updateTemperatureHumidity();

    // Publish the updated values to AWS
    publishMessage();
  }

  // Check if the current time is 21:21:00
  time_t now = time(nullptr);
  struct tm *timeinfo = localtime(&now);
  if (timeinfo->tm_hour == 23 && timeinfo->tm_min == 30 && timeinfo->tm_sec == 0) {
    alarmActive = true;
  } else {
    alarmActive = false;
  }

  // Control the buzzer based on alarm status
  if (alarmActive) {
    digitalWrite(BUZZER_PIN, HIGH); // Set buzzer to HIGH continuously
    Serial.println("Alarm Active");
  } else {
    digitalWrite(BUZZER_PIN, LOW); // Turn off buzzer when alarm is inactive
  }

  // Buzzer control with button press
  if (buttonState == LOW) { // Button is pressed
    if (!isButtonPressed) {
      isButtonPressed = true;
      buttonPressTime = millis();
    } else if (millis() - buttonPressTime >= debounceDelay) { // Button is held long enough
      if (alarmActive) {
        // Stop the buzzer and reset alarmActive
        digitalWrite(BUZZER_PIN, LOW);
        alarmActive = false;
        Serial.println("Alarm Stopped");
      }
      isButtonPressed = false; // Reset after button release
    }
  }

  display.display();
  delay(100); // Small delay to avoid flickering
}

void updateTemperatureHumidity() {
  h = dht.readHumidity();
  t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println(F("Failed to read from DHT sensor!"));
    return;
  }

  if (showTemperature) {
    displayTemperatureHumidity();
  }
}

void displayTime() {
  time_t now = time(nullptr);
  struct tm *timeinfo = localtime(&now);

  char timeString[9]; // Buffer to store formatted time string
  sprintf(timeString, "%02d:%02d:%02d", timeinfo->tm_hour, timeinfo->tm_min, timeinfo->tm_sec);

  int16_t x1, y1;
  uint16_t width, height;
  
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.getTextBounds(timeString, 0, 0, &x1, &y1, &width, &height);
  
  display.setCursor((SCREEN_WIDTH - width) / 2, (SCREEN_HEIGHT - height) / 2);
  display.print(timeString);
  display.display();
}

void displayTemperatureHumidity() {
  display.clearDisplay();
  display.setTextSize(1);  // Keep the text size at 1
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(0, 0);
  display.print("Temp: ");
  display.print(t);
  display.print(" C");
  
  display.setCursor(0, 20);
  display.print("Humidity: ");
  display.print(h);
  display.print(" %");

  display.display();
}

void displayMessage() {
  display.clearDisplay();
  display.setTextSize(1);  // Set the text size
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Message:");
  display.setCursor(0, 20);
  display.println(awsMessage);
  display.display();
}

void displayLoadingScreen() {
  static uint8_t frameIndex = 0;
  static unsigned long lastFrameTime = 0;

  if (millis() - lastFrameTime > FRAME_DELAY) {
    lastFrameTime = millis();
    display.clearDisplay();

    // Calculate the horizontal position to center the frame
    int xPosition = (SCREEN_WIDTH - FRAME_WIDTH) / 2;

    display.drawBitmap(xPosition, 0, frames[frameIndex], FRAME_WIDTH, FRAME_HEIGHT, WHITE);
    display.display();

    frameIndex = (frameIndex + 1) % FRAME_COUNT;
  }
}



SECRETS.h

#include <pgmspace.h>
 
#define SECRET
 
const char WIFI_SSID[] = "Shiv_2.4Ghz";               //TAMIM2.4G
const char WIFI_PASSWORD[] = "shiv0000";           //0544287380
 
#define THINGNAME "ESP8266"
 
int8_t TIME_ZONE = -5; //NYC(USA): -5 UTC
 
const char MQTT_HOST[] = "a13r91rrfe597k-ats.iot.ap-south-1.amazonaws.com";
 
 
static const char cacert[] PROGMEM = R"EOF(
-----BEGIN CERTIFICATE-----
MIIDQTCCAimgAwIBAgITBmyfz5m/jAo54vB4ikPmljZbyjANBgkqhkiG9w0BAQsF
ADA5MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRkwFwYDVQQDExBBbWF6
b24gUm9vdCBDQSAxMB4XDTE1MDUyNjAwMDAwMFoXDTM4MDExNzAwMDAwMFowOTEL
MAkGA1UEBhMCVVMxDzANBgNVBAoTBkFtYXpvbjEZMBcGA1UEAxMQQW1hem9uIFJv
b3QgQ0EgMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALJ4gHHKeNXj
ca9HgFB0fW7Y14h29Jlo91ghYPl0hAEvrAIthtOgQ3pOsqTQNroBvo3bSMgHFzZM
9O6II8c+6zf1tRn4SWiw3te5djgdYZ6k/oI2peVKVuRF4fn9tBb6dNqcmzU5L/qw
IFAGbHrQgLKm+a/sRxmPUDgH3KKHOVj4utWp+UhnMJbulHheb4mjUcAwhmahRWa6
VOujw5H5SNz/0egwLX0tdHA114gk957EWW67c4cX8jJGKLhD+rcdqsq08p8kDi1L
93FcXmn/6pUCyziKrlA4b9v7LWIbxcceVOF34GfID5yHI9Y/QCB/IIDEgEw+OyQm
jgSubJrIqg0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMC
AYYwHQYDVR0OBBYEFIQYzIU07LwMlJQuCFmcx7IQTgoIMA0GCSqGSIb3DQEBCwUA
A4IBAQCY8jdaQZChGsV2USggNiMOruYou6r4lK5IpDB/G/wkjUu0yKGX9rbxenDI
U5PMCCjjmCXPI6T53iHTfIUJrU6adTrCC2qJeHZERxhlbI1Bjjt/msv0tadQ1wUs
N+gDS63pYaACbvXy8MWy7Vu33PqUXHeeE6V/Uq2V8viTO96LXFvKWlJbYK8U90vv
o/ufQJVtMVT8QtPHRh8jrdkPSHCa2XV4cdFyQzR1bldZwgJcJmApzyMZFo6IQ6XU
5MsI+yMRQ+hDKXJioaldXgjUkK642M4UwtBV8ob2xJNDd2ZhwLnoQdeXeGADbkpy
rqXRfboQnoZsG4q5WTP468SQvvG5
-----END CERTIFICATE-----
)EOF";
 
 
// Copy contents from XXXXXXXX-certificate.pem.crt here ▼
static const char client_cert[] PROGMEM = R"KEY(
-----BEGIN CERTIFICATE-----
MIIDWjCCAkKgAwIBAgIVAPLMJ+tpMnyu9o3cBU2MH65DKgrVMA0GCSqGSIb3DQEB
CwUAME0xSzBJBgNVBAsMQkFtYXpvbiBXZWIgU2VydmljZXMgTz1BbWF6b24uY29t
IEluYy4gTD1TZWF0dGxlIFNUPVdhc2hpbmd0b24gQz1VUzAeFw0yNDA4MjkwOTA5
MjZaFw00OTEyMzEyMzU5NTlaMB4xHDAaBgNVBAMME0FXUyBJb1QgQ2VydGlmaWNh
dGUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCfPaF+axB/KK7w77oX
J0TFyEUeR1oZnhycT/MX5XpvcZTeoXAU6Wy7RSlH+TAU9LjyC8TMr+7O3QGsgh8x
MXub+Z88D/jKVKlaDwh75dfyyNzBKElKASRj+W6nrPNH8LSCd+OWS8L6rEHFNlXg
Zuhg8EHDp6Db+7JacAmPpu6kbHhWqlyUfBfDnLPrq07JQL1ujvMpb8fWM8cECu0r
4PDaPqqdG85gCHAUPAxYAWMdWg9nemDcuM9bC6dj2XR6xihX5UajhsFfca8yD6UF
Bo7+4fcWlXtM36toJ2nYFrNY6I4ShIvavPNpvyiHQni5e+qIYAoGne/Rn/4jw8YQ
mmifAgMBAAGjYDBeMB8GA1UdIwQYMBaAFJg3X2LqN4dHNAHjNnlEBmAtG2dGMB0G
A1UdDgQWBBQOdLt3jchcq7ZEgeXSbH7BVBdoHjAMBgNVHRMBAf8EAjAAMA4GA1Ud
DwEB/wQEAwIHgDANBgkqhkiG9w0BAQsFAAOCAQEANU1hyeToydgSHpuIcCCqL7Qr
ufMUSG4J8mRAstCzeKDbtqgJj/0Gy/eybslZ0e08aWwZfIZkCv66jT3sfdRgSanH
wFH4NzVsLZmU9IAFZxrBMZSjnp38KGbv9GgwHP+5hhao/vYy6mxKHWZ0epdRMuRR
gi9ceSl/nnmLvtqCBvsKjxS0FuTMOq3vXZh0g8rhlPwfeQDhH4USu51bVa1NeZl3
JOykSJLI/e9y4U+m6Gw+sipVk7T0VOx5qLZmPdLEqyET+pGPoQStOYZzG1KlmcG3
95KPf/yzyW9Cisnc2Akdgj/IQrBzVKquTk4mvyLU78At4GHoGQ5E1D5i69hIUA==
-----END CERTIFICATE-----
 
)KEY";
 
 
// Copy contents from  XXXXXXXX-private.pem.key here ▼
static const char privkey[] PROGMEM = R"KEY(
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAnz2hfmsQfyiu8O+6FydExchFHkdaGZ4cnE/zF+V6b3GU3qFw
FOlsu0UpR/kwFPS48gvEzK/uzt0BrIIfMTF7m/mfPA/4ylSpWg8Ie+XX8sjcwShJ
SgEkY/lup6zzR/C0gnfjlkvC+qxBxTZV4GboYPBBw6eg2/uyWnAJj6bupGx4Vqpc
lHwXw5yz66tOyUC9bo7zKW/H1jPHBArtK+Dw2j6qnRvOYAhwFDwMWAFjHVoPZ3pg
3LjPWwunY9l0esYoV+VGo4bBX3GvMg+lBQaO/uH3FpV7TN+raCdp2BazWOiOEoSL
2rzzab8oh0J4uXvqiGAKBp3v0Z/+I8PGEJponwIDAQABAoIBAQCE69wO+23Exv/o
bCMoyoWUlsxjLuodsiZtsCrZypq9xdCfWaCGRCaX125S/sVM6M4sdPhsZ3ruv/py
thc1Z/mnQ+HQMADbW3oVi7DoQv5UUag7r9YlaPioXwAoBKz6Ywk6UrrtrQXvWrR3
2xgp/ZyBtmse16Dln57L8PN6LrzLD9L1AgIkb/MHyczblV9Er7yOvtF2uuxCAcAg
8vZgcSNKxrZW/yYbgtJLgbXF7H5gNuUkNe2BbGq3nLVKYLnzsuiNNDa5FJ44nMS3
bDVmrbk92ehD0XaCvZ2/buFmPDn6flWzyitRk7+i2lBALRPFGm5ZjKJK9hvTY1tV
34ydjuSRAoGBAMyhG71sUnJHWKA8RibFr0+JEoQinYpInTPkEbiYRkQIgZql4hQc
Ahuc34oOStBW+xdskC6MfnTTC6tRHEGVOpIWSlexd+6vmAk8fOszfKrl7ECwbAJY
0iWJMJhhImQQWsAK9K0UUzFaw4ayVaiMlhdA1sOuDIsl52UCI0Sljr7ZAoGBAMc3
igL1JQ7lSY3Brp9slrKKNmbeEa/nvKECxHIOTVX4XA/oABVPT/cPGoBQmbGf3Za+
998th1tGcgrH6odsUKVINsZ4S+7d3X0dS1kZofamTGcCmpP1Z5b785vYQVmGFvzZ
S4E4hD+etYkG/ueqlmB5SlDJxEV27w+pOGGSmag3AoGAUtOXZdnVmVoVnm4nOwRj
TH9AFmnoeJOhxeI35g8Eyf7jbtRcKSWZGNIrjTbxw1ihs76GscC+Ys0V+RcQp98e
YQlSuCImWF+M25g3PACQIqCEOz7tyRlonjbki5ktkXEpOnh0xyXl8qE5aWj/0QRu
sCTXiUcG3r/N5I2z9tJIcCkCgYEAxOXJzF6LAAvzBN63Pu7Oiyw71LQL+zYpo2He
03P7T8snArmky2sWd/M/mC8RmROOqZ2Z08VmEPqxYKJy1OJjWtji+oqPUkmKzkwT
2r6Q6/01amKScUaN2havkgrNnDQBqGsES3WWkGLGveZiLorWEggPQYYKLTX91hbE
mPuST0UCgYA7xC/j8+RRvEkw/x00mVGqR2CHHwMPb88N2FvI9Ag7rNws/0NEN4hR
Lt3ZZj7DgcgUBBnWUm/Q/6u4rxZqx5tjKl19FYftgbu23dCiEFvJ5ov/wBIEa7MZ
pcBxjqt3QC/AM55EzMNTgWzTRq3WbEuxvYx80osxHcJ8vtFYqXKQ4w==
-----END RSA PRIVATE KEY-----
 
)KEY";
