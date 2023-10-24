i am nuwan wasantha karunarathna i am from sri lanaka 



#include <WiFi.h>

#include <PubSubClient.h>
#include <Adafruit_Fingerprint.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Define the GPIO pin for D5 (you can use any available GPIO pin)
const int ledPin = 5;

// Update these with values suitable for your network and MQTT broker.
const char* ssid = "nuwan";          // Your Wi-Fi network SSID
const char* password = "nuwannnnn";  // Your Wi-Fi network password
const char* mqtt_server = "91.121.93.94"; // MQTT broker server IP address
const char* fingerprint_topic = "device/fingerprint"; // MQTT topic for fingerprint status

WiFiClient espClient;
PubSubClient client(espClient);

#define mySerial Serial2 // Use Serial2 for fingerprint sensor

Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);

LiquidCrystal_I2C lcd(0x27, 16, 2); // I2C address (0x27) and dimensions (16x2)

void setup_wifi() {
  // We start by connecting to a WiFi network
  Serial.begin(115200);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  if ((char)payload[0] == '1') {
    digitalWrite(ledPin, LOW);
  } else {
    digitalWrite(ledPin, HIGH);
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-";
    clientId += String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
      client.publish("device/pot", "MQTT Server Connected!");
      client.subscribe("device/led");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}

void setup() {
  pinMode(ledPin, OUTPUT); // Set D5 (or your chosen GPIO pin) as an output for the LED
  digitalWrite(ledPin, HIGH); // Initialize the LED as OFF
  Serial.begin(115200);  // Initialize serial communication
  setup_wifi();  // Connect to Wi-Fi
  client.setServer(mqtt_server, 1883); // Configure MQTT client
  client.setCallback(callback);  // Set the callback function for incoming messages

  mySerial.begin(57600);

  if (!finger.verifyPassword()) {
    Serial.println("Could not find a fingerprint sensor :(");
    while (1);
  }

  Serial.println("Found fingerprint sensor!");

  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Fingerprint Sensor");
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  uint8_t p = getFingerprintID();

  if (p == FINGERPRINT_OK) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Fingerprint Identified");
    client.publish(fingerprint_topic, "Fingerprint Identified");
    digitalWrite(ledPin, LOW); // Turn on the LED
    delay(2000);
  } else if (p == FINGERPRINT_NOTFOUND) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Fingerprint Not Identified");
    client.publish(fingerprint_topic, "Fingerprint Not Identified");
    digitalWrite(ledPin, HIGH); // Turn off the LED
    delay(2000);
  }
  lcd.clear();
}

uint8_t getFingerprintID() {
  uint8_t p = -1;
  mySerial.begin(57600);

  while (!mySerial.available()) {
    // Wait for the fingerprint sensor to become available
  }

  p = finger.getImage();

  switch (p) {
    case FINGERPRINT_OK:
      // Your fingerprint identification logic here
      break;
    case FINGERPRINT_NOFINGER:
      break;
    case FINGERPRINT_PACKETRECIEVEERR:
      break;
    case FINGERPRINT_IMAGEFAIL:
      break;
    default:
      break;
  }

  return p;
}
