# smart-home-from-sinric-pro
#include <WiFi.h>
#include <SinricPro.h>
#include <SinricProSwitch.h>
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ========== WiFi Credentials ==========
#define WIFI_SSID         "WIFI SISD NAME"
#define WIFI_PASS         "WIFI PASSWORD"

// ========== Sinric Pro Credentials ==========
#define APP_KEY           "ded237dc-a286-495e-8cb9-4499259194ff"
#define APP_SECRET        "961af354-bc5f-4b65-a37d-7375b4b7920b-e08644c3-131a-4846-8535-abd152ec41c6"

// ========== Sinric Pro Device IDs ==========
#define MODE_SWITCH_ID    "6959f5b7971001dc558b1b8f"    // Virtual Mode Switch
#define FAN_SWITCH_ID     "695697e1971001dc55899c67"     // Fan Device
#define LIGHT_SWITCH_ID   "69575f86fd289c6add4a370b"     // Light Device

// ========== Pin Definitions ==========
// MOSFET Driver Modules (Active HIGH)
#define FAN_MOSFET_PIN    26    // GPIO26 - Fan MOSFET Gate Control
#define LIGHT_MOSFET_PIN  27    // GPIO27 - Light MOSFET Gate Control

// Sensors
#define LDR_PIN           34    // GPIO34 - LDR Digital Output (INPUT ONLY)
#define DHT_PIN           25    // GPIO25 - DHT22 Data Pin
#define OLED_SDA          21    // GPIO21 - I2C SDA (default)
#define OLED_SCL          22    // GPIO22 - I2C SCL (default)
#define IR_IN             32    // GPIO32 - Entry IR sensor pin
#define IR_EXIT           33    // GPIO33 - Exit IR sensor pin

// ========== DHT22 Configuration ==========
#define DHTTYPE           DHT22
DHT dht(DHT_PIN, DHTTYPE);

// ========== OLED Configuration (128x32) ==========
#define SCREEN_WIDTH      128
#define SCREEN_HEIGHT     32
#define OLED_RESET        -1    // Reset pin (or -1 if sharing ESP32 reset pin)
#define OLED_ADDRESS      0x3C  // Common I2C address for 128x32 OLED
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ========== Sensor Thresholds ==========
#define TEMP_THRESHOLD    28.0  // Temperature threshold in °C for Fan
#define LDR_DARK          HIGH  // LDR output when dark (depends on module)

// ========== Timing Constants ==========
#define SENSOR_READ_INTERVAL  2000  // Read sensors every 2 seconds
#define DISPLAY_UPDATE_INTERVAL 500 // Update display every 500ms
#define RECONNECT_INTERVAL    60000 // WiFi reconnect check every 60s
#define IR_DEBOUNCE_DELAY     50    // IR sensor debounce delay in ms

// ========== Global Variables ==========
bool isAutoMode = false;           // false = Manual Mode, true = Auto Mode
bool fanState = false;             // Current fan state
bool lightState = false;           // Current light state

unsigned long lastSensorRead = 0;
unsigned long lastDisplayUpdate = 0;
unsigned long lastReconnect = 0;

// Person Counter Variables
int personCount = 0;               // Number of people inside
bool entryTriggered = false; 
bool exitTriggered = false;
unsigned long lastIRCheck = 0;

// DHT22 Variables
bool dhtInitialized = false;
float currentTemperature = 0.0;
float currentHumidity = 0.0;

// OLED Display Variables (to prevent unnecessary updates)
int lastDisplayedCount = -1;
bool lastDisplayedFanState = false;
float lastDisplayedTemp = -999.0;

// ========== Function Prototypes ==========
void setupWiFi();
void setupSinricPro();
void setupSensors();
void setupOLED();
void updateOLED();
void controlFan(bool state);
void controlLight(bool state);
void readSensorsAutoMode();
void checkPersonCounter();
bool onModeSwitch(const String &deviceId, bool &state);
bool onFanSwitch(const String &deviceId, bool &state);
bool onLightSwitch(const String &deviceId, bool &state);

// ========== SETUP ==========
void setup() {
  Serial.begin(115200);
  while(!Serial) delay(10);
  
  Serial.println("\n\n========================================");
  Serial.println("ESP32 Smart Home Automation System");
  Serial.println("with DHT22 + OLED Display");
  Serial.println("MOSFET Driver Module Version");
  Serial.println("========================================\n");

  // Initialize pins
  pinMode(FAN_MOSFET_PIN, OUTPUT);
  pinMode(LIGHT_MOSFET_PIN, OUTPUT);
  pinMode(LDR_PIN, INPUT);
  pinMode(IR_IN, INPUT);
  pinMode(IR_EXIT, INPUT);
  
  // Ensure MOSFETs are OFF at startup (Active HIGH, so LOW = OFF)
  digitalWrite(FAN_MOSFET_PIN, LOW);
  digitalWrite(LIGHT_MOSFET_PIN, LOW);
  
  Serial.println("✓ GPIO pins configured");
  Serial.println("✓ MOSFET modules initialized (OFF state)");
  Serial.println("✓ Person Counter initialized");

  // Setup components
  setupWiFi();
  setupSensors();
  setupOLED();
  setupSinricPro();

  Serial.println("\n========================================");
  Serial.println("System Ready - Starting in MANUAL MODE");
  Serial.println("Person Count: 0");
  Serial.println("========================================\n");
  
  // Initial display update
  updateOLED();
}

// ========== MAIN LOOP ==========
void loop() {
  // Handle Sinric Pro communication
  SinricPro.handle();

  // Check person counter (always active, but only affects Auto Mode)
  checkPersonCounter();

  // Auto Mode: Read sensors and control devices
  if (isAutoMode) {
    unsigned long currentMillis = millis();
    
    if (currentMillis - lastSensorRead >= SENSOR_READ_INTERVAL) {
      lastSensorRead = currentMillis;
      readSensorsAutoMode();
    }
  }

  // Update OLED display periodically
  if (millis() - lastDisplayUpdate >= DISPLAY_UPDATE_INTERVAL) {
    lastDisplayUpdate = millis();
    updateOLED();
  }

  // WiFi reconnection check
  if (millis() - lastReconnect >= RECONNECT_INTERVAL) {
    lastReconnect = millis();
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("⚠ WiFi disconnected! Reconnecting...");
      setupWiFi();
    }
  }
}

// ========== WiFi Setup ==========
void setupWiFi() {
  Serial.print("Connecting to WiFi: ");
  Serial.println(WIFI_SSID);
  
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✓ WiFi Connected!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\n✗ WiFi Connection Failed!");
  }
}

// ========== Sensor Setup ==========
void setupSensors() {
  Serial.println("\nInitializing Sensors...");
  
  // Initialize DHT22
  dht.begin();
  delay(2000); // DHT22 needs time to stabilize
  
  // Test DHT22 reading
  float testTemp = dht.readTemperature();
  float testHum = dht.readHumidity();
  
  if (!isnan(testTemp) && !isnan(testHum)) {
    dhtInitialized = true;
    Serial.println("✓ DHT22 sensor initialized");
    Serial.print("  Initial Temperature: ");
    Serial.print(testTemp, 1);
    Serial.println(" °C");
    Serial.print("  Initial Humidity: ");
    Serial.print(testHum, 1);
    Serial.println(" %");
  } else {
    dhtInitialized = false;
    Serial.println("✗ DHT22 sensor not found or not responding!");
    Serial.println("  Check wiring and connections");
  }
  
  // LDR is passive input - no initialization needed
  Serial.println("✓ LDR module ready");
  Serial.println("✓ IR sensors ready");
}

// ========== OLED Setup ==========
void setupOLED() {
  Serial.println("\nInitializing OLED Display...");
  
  // Initialize I2C for OLED
  Wire.begin(OLED_SDA, OLED_SCL);
  
  // Initialize OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("✗ OLED display allocation failed!");
    Serial.println("  Check wiring and I2C address (0x3C or 0x3D)");
    // Continue anyway - system can work without display
  } else {
    Serial.println("✓ OLED display initialized at 0x3C");
    
    // Clear display and show startup message
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println("Smart Home");
    display.println("Initializing...");
    display.display();
    delay(1000);
  }
}

// ========== OLED Update Function (Optimized for 128x32) ==========
void updateOLED() {
  // Check if any values have changed (prevent unnecessary updates)
  bool needsUpdate = false;
  
  if (lastDisplayedCount != personCount ||
      lastDisplayedFanState != fanState ||
      abs(lastDisplayedTemp - currentTemperature) > 0.1) {
    needsUpdate = true;
  }
  
  if (!needsUpdate) {
    return; // No changes, skip update
  }
  
  // Update stored values
  lastDisplayedCount = personCount;
  lastDisplayedFanState = fanState;
  lastDisplayedTemp = currentTemperature;
  
  // Clear display
  display.clearDisplay();
  
  // ===== Line 1: Mode + Person Count =====
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print(isAutoMode ? "AUTO" : "MAN");
  display.print(" | People:");
  display.print(personCount);
  
  // ===== Line 2: Fan Status =====
  display.setCursor(0, 10);
  display.print("Fan:");
  display.print(fanState ? "ON " : "OFF");
  
  // ===== Line 3: Temperature =====
  display.setCursor(0, 20);
  if (dhtInitialized && !isnan(currentTemperature)) {
    display.print("Temp:");
    display.print(currentTemperature, 1);
    display.print("C");
  } else {
    display.print("Temp:--.-C");
  }
  
  // ===== Right side: Humidity (if space allows) =====
  display.setCursor(70, 20);
  if (dhtInitialized && !isnan(currentHumidity)) {
    display.print("H:");
    display.print(currentHumidity, 0);
    display.print("%");
  } else {
    display.print("H:--%");
  }
  
  // Update display
  display.display();
}

// ========== Sinric Pro Setup ==========
void setupSinricPro() {
  Serial.println("\nConfiguring Sinric Pro...");

  // Setup Mode Switch (Virtual Switch)
  SinricProSwitch& modeSwitch = SinricPro[MODE_SWITCH_ID];
  modeSwitch.onPowerState(onModeSwitch);

  // Setup Fan Switch
  SinricProSwitch& fanSwitch = SinricPro[FAN_SWITCH_ID];
  fanSwitch.onPowerState(onFanSwitch);

  // Setup Light Switch
  SinricProSwitch& lightSwitch = SinricPro[LIGHT_SWITCH_ID];
  lightSwitch.onPowerState(onLightSwitch);

  // Connect to Sinric Pro
  SinricPro.begin(APP_KEY, APP_SECRET);
  
  Serial.println("✓ Sinric Pro configured");
}

// ========== Person Counter Function ==========
void checkPersonCounter() {
  unsigned long currentMillis = millis();
  
  // Debounce IR sensor readings
  if (currentMillis - lastIRCheck < IR_DEBOUNCE_DELAY) {
    return;
  }
  lastIRCheck = currentMillis;
  
  // Read IR sensor states
  int entryState = digitalRead(IR_IN);      // HIGH or LOW
  int exitState = digitalRead(IR_EXIT);
  
  // Entry detection
  if (entryState == HIGH && !entryTriggered) { // IR blocked, person enters
    personCount++;
    entryTriggered = true;
    Serial.println("\n╔════════════════════════════════════╗");
    Serial.print("║  👤 ENTRY DETECTED → Count: ");
    Serial.print(personCount);
    Serial.println("      ║");
    Serial.println("╚════════════════════════════════════╝");
  }
  if (entryState == LOW) {   // Reset trigger when no one blocking
    entryTriggered = false;
  }
  
  // Exit detection
  if (exitState == HIGH && !exitTriggered) {  // IR blocked, person exits
    if (personCount > 0) personCount--;       // avoid negative count
    exitTriggered = true;
    Serial.println("\n╔════════════════════════════════════╗");
    Serial.print("║  👤 EXIT DETECTED → Count: ");
    Serial.print(personCount);
    Serial.println("       ║");
    Serial.println("╚════════════════════════════════════╝");
    
    // Turn OFF all devices when count reaches 0 in Auto Mode
    if (personCount == 0 && isAutoMode) {
      Serial.println("\n⚠ NO PEOPLE INSIDE - Turning OFF all devices");
      controlFan(false);
      controlLight(false);
    }
  }
  if (exitState == LOW) {    // Reset trigger when no one blocking
    exitTriggered = false;
  }
}

// ========== MOSFET Control Functions ==========
// MOSFET modules are Active HIGH: HIGH = ON, LOW = OFF

void controlFan(bool state) {
  if (fanState != state) {
    fanState = state;
    digitalWrite(FAN_MOSFET_PIN, state ? HIGH : LOW);
    Serial.print("🌀 FAN turned ");
    Serial.println(state ? "ON" : "OFF");
    
    // Update Sinric Pro state
    SinricProSwitch& fanSwitch = SinricPro[FAN_SWITCH_ID];
    fanSwitch.sendPowerStateEvent(state);
  }
}

void controlLight(bool state) {
  if (lightState != state) {
    lightState = state;
    digitalWrite(LIGHT_MOSFET_PIN, state ? HIGH : LOW);
    Serial.print("💡 LIGHT turned ");
    Serial.println(state ? "ON" : "OFF");
    
    // Update Sinric Pro state
    SinricProSwitch& lightSwitch = SinricPro[LIGHT_SWITCH_ID];
    lightSwitch.sendPowerStateEvent(state);
  }
}

// ========== Sensor Reading (Auto Mode Only) ==========
void readSensorsAutoMode() {
  // If no people inside, keep everything OFF
  if (personCount == 0) {
    Serial.println("\n--- AUTO MODE: No people inside ---");
    Serial.println("⚠ All devices kept OFF (Person Count = 0)");
    controlFan(false);
    controlLight(false);
    Serial.println("----------------------------------");
    return;
  }
  
  Serial.println("\n--- AUTO MODE: Reading Sensors ---");
  Serial.print("👥 People inside: ");
  Serial.println(personCount);
  
  // Read DHT22 Temperature and Humidity
  if (dhtInitialized) {
    float temperature = dht.readTemperature();
    float humidity = dht.readHumidity();
    
    // Validate readings
    if (!isnan(temperature) && !isnan(humidity)) {
      currentTemperature = temperature;
      currentHumidity = humidity;
      
      Serial.print("🌡 Temperature: ");
      Serial.print(temperature, 1);
      Serial.println(" °C");
      
      Serial.print("💧 Humidity: ");
      Serial.print(humidity, 1);
      Serial.println(" %");
      
      // Fan control based on temperature
      if (temperature > TEMP_THRESHOLD) {
        controlFan(true);
        Serial.println("   → Fan ON (Temperature exceeded threshold)");
      } else {
        controlFan(false);
        Serial.println("   → Fan OFF (Temperature below threshold)");
      }
    } else {
      Serial.println("⚠ DHT22 read error (NaN values)");
    }
  } else {
    Serial.println("⚠ DHT22 not available - Fan control disabled");
  }
  
  // Read LDR (Digital Output)
  int ldrValue = digitalRead(LDR_PIN);
  
  if (ldrValue == LDR_DARK) {
    Serial.println("🌙 LDR: DARK detected");
    controlLight(true);
    Serial.println("   → Light ON");
  } else {
    Serial.println("☀ LDR: BRIGHT detected");
    controlLight(false);
    Serial.println("   → Light OFF");
  }
  
  Serial.println("----------------------------------");
}

// ========== Sinric Pro Callbacks ==========

// Mode Switch Callback
bool onModeSwitch(const String &deviceId, bool &state) {
  isAutoMode = state;
  
  Serial.println("\n╔════════════════════════════════════╗");
  Serial.print("║  MODE CHANGED: ");
  Serial.print(state ? "AUTO MODE     " : "MANUAL MODE   ");
  Serial.println("║");
  Serial.println("╚════════════════════════════════════╝");
  
  if (isAutoMode) {
    Serial.println("✓ Sensors activated - Google Assistant commands will be IGNORED");
    Serial.print("👥 Current person count: ");
    Serial.println(personCount);
    
    // If no people inside, turn everything OFF immediately
    if (personCount == 0) {
      Serial.println("⚠ No people inside - Turning OFF all devices");
      controlFan(false);
      controlLight(false);
    }
    
    lastSensorRead = 0; // Force immediate sensor read
  } else {
    Serial.println("✓ Manual control enabled - Sensors IGNORED");
    Serial.println("✓ Person counter still tracking (affects Auto Mode only)");
  }
  
  return true;
}

// Fan Switch Callback
bool onFanSwitch(const String &deviceId, bool &state) {
  if (isAutoMode) {
    Serial.println("✗ REJECTED: Fan control command (Auto Mode active)");
    Serial.println("  → Change to Manual Mode to control via Google Assistant");
    return true; // Accept but don't execute
  }
  
  Serial.print("📱 Manual command received: Fan ");
  Serial.println(state ? "ON" : "OFF");
  controlFan(state);
  return true;
}

// Light Switch Callback
bool onLightSwitch(const String &deviceId, bool &state) {
  if (isAutoMode) {
    Serial.println("✗ REJECTED: Light control command (Auto Mode active)");
    Serial.println("  → Change to Manual Mode to control via Google Assistant");
    return true; // Accept but don't execute
  }
  
  Serial.print("📱 Manual command received: Light ");
  Serial.println(state ? "ON" : "OFF");
  controlLight(state);
  return true;
}
