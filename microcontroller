#include <WiFi.h>
#include <WebServer.h>
#include <ArduinoJson.h>
#include <Preferences.h>

// Network Configuration
const char* WIFI_SSID = "PNRENGIOT";
const char* WIFI_PASSWORD = "WP3MQJCsqYqstXm5";

// Pin definitions for ESP32-C6-DevKitC-1
const int PRESSURE_PINS[3] = {0, 1, 2};     // Analog pins for pressure transducers (0.534V-3.0V) - ADC1_CH0, ADC1_CH1, ADC1_CH2
const int FLOW_PINS[3] = {18, 19, 20};      // Digital pins for flow meter pulse inputs
                                             // Flow meter wiring: Red(VCC+) to 5V, Black(GND-) to GND, Yellow(Signal) to Pin

// Flow meter variables
volatile int pulseCounts[3] = {0, 0, 0};
float flowRates[3] = {0.0, 0.0, 0.0};
float frequencies[3] = {0.0, 0.0, 0.0};
unsigned long previousMillis = 0;
const unsigned long FLOW_CALC_INTERVAL = 1000; // Calculate flow every second

// Flow meter calibration factors - configurable via web interface
float flowCalibrationFactors[3] = {21.0, 21.0, 21.0}; // Default to small flow meters
float flowMinLimits[3] = {0.3, 0.3, 0.3};  // Minimum flow rates (L/min)
float flowMaxLimits[3] = {10.0, 10.0, 10.0}; // Maximum flow rates (L/min)

// Pressure variables
float pressures[3] = {0.0, 0.0, 0.0};
float pressureVoltages[3] = {0.0, 0.0, 0.0};
// Pressure calibration - configurable via web interface
float pressureMinLimits[3] = {0.0, 0.0, 0.0};    // Minimum pressure (PSI)
float pressureMaxLimits[3] = {100.0, 100.0, 100.0}; // Maximum pressure (PSI)

// Web server
WebServer server(80);

// Preferences for storing configuration
Preferences preferences;

// Flow meter interrupt functions
void IRAM_ATTR pulseCounter0() { pulseCounts[0]++; }
void IRAM_ATTR pulseCounter1() { pulseCounts[1]++; }
void IRAM_ATTR pulseCounter2() { pulseCounts[2]++; }

void setup() {
  Serial.begin(115200);
  
  // Initialize preferences
  preferences.begin("water_monitor", false);
  loadConfiguration();
  
  // Initialize pins
  for (int i = 0; i < 3; i++) {
    pinMode(FLOW_PINS[i], INPUT_PULLUP);
    pinMode(PRESSURE_PINS[i], INPUT);
  }
  
  // Attach interrupts for flow meters
  attachInterrupt(digitalPinToInterrupt(FLOW_PINS[0]), pulseCounter0, FALLING);
  attachInterrupt(digitalPinToInterrupt(FLOW_PINS[1]), pulseCounter1, FALLING);
  attachInterrupt(digitalPinToInterrupt(FLOW_PINS[2]), pulseCounter2, FALLING);
  
  // Connect to WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println();
  Serial.print("Connected! IP address: ");
  Serial.println(WiFi.localIP());
  
  // Setup web server routes
  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.on("/config", HTTP_GET, handleGetConfig);
  server.on("/config", HTTP_POST, handleSetConfig);
  server.enableCORS(true);
  
  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
  
  // Calculate flow rates every second
  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= FLOW_CALC_INTERVAL) {
    for (int i = 0; i < 3; i++) {
      // Calculate frequency (Hz) = pulses per second
      frequencies[i] = pulseCounts[i]; // pulses counted in 1 second
      
      // Calculate flow rate using calibration formula: Q = F / calibration_factor
      // Small flow meter: F = 21 * Q, so Q = F / 21
      // Big flow meter: F = 7.5 * Q, so Q = F / 7.5
      flowRates[i] = frequencies[i] / flowCalibrationFactors[i];
      
      // Apply working range limits
      if (flowRates[i] < flowMinLimits[i] && flowRates[i] > 0.05) {
        flowRates[i] = 0.0; // Below minimum measurable range
      } else if (flowRates[i] > flowMaxLimits[i]) {
        flowRates[i] = flowMaxLimits[i]; // Cap at maximum range
      }
      
      pulseCounts[i] = 0; // Reset pulse count
    }
    previousMillis = currentMillis;
  }
  
  // Read pressure sensors
  for (int i = 0; i < 3; i++) {
    int adcValue = analogRead(PRESSURE_PINS[i]);
    // Convert ADC value to voltage (ESP32 ADC is 12-bit, 0-4095 for 0-3.3V)
    pressureVoltages[i] = (adcValue * 3.3) / 4095.0; // ESP32 native voltage range
    
    // Convert voltage to pressure using 0.534V - 3.0V range
    // Clamp voltage to valid sensor range first
    float clampedVoltage = pressureVoltages[i];
    if (clampedVoltage < 0.534) clampedVoltage = 0.534;
    if (clampedVoltage > 3.0) clampedVoltage = 3.0;
    
    // Linear mapping: voltage (0.534V-3.0V) to pressure (min-max PSI)
    float voltageRange = 3.0 - 0.534; // 2.466V range
    float normalizedVoltage = (clampedVoltage - 0.534) / voltageRange; // 0.0 to 1.0
    pressures[i] = pressureMinLimits[i] + (normalizedVoltage * (pressureMaxLimits[i] - pressureMinLimits[i]));
    
    // Final clamp to configured pressure limits
    if (pressures[i] < pressureMinLimits[i]) pressures[i] = pressureMinLimits[i];
    if (pressures[i] > pressureMaxLimits[i]) pressures[i] = pressureMaxLimits[i];
  }
  
  delay(50); // Small delay for stability
}

void loadConfiguration() {
  // Load flow meter configurations
  for (int i = 0; i < 3; i++) {
    String factorKey = "flow_factor_" + String(i);
    String minKey = "flow_min_" + String(i);
    String maxKey = "flow_max_" + String(i);
    
    flowCalibrationFactors[i] = preferences.getFloat(factorKey.c_str(), 21.0);
    flowMinLimits[i] = preferences.getFloat(minKey.c_str(), 0.3);
    flowMaxLimits[i] = preferences.getFloat(maxKey.c_str(), 10.0);
  }
  
  // Load pressure configurations
  for (int i = 0; i < 3; i++) {
    String minKey = "press_min_" + String(i);
    String maxKey = "press_max_" + String(i);
    
    pressureMinLimits[i] = preferences.getFloat(minKey.c_str(), 0.0);
    pressureMaxLimits[i] = preferences.getFloat(maxKey.c_str(), 100.0);
  }
}

void saveConfiguration() {
  // Save flow meter configurations
  for (int i = 0; i < 3; i++) {
    String factorKey = "flow_factor_" + String(i);
    String minKey = "flow_min_" + String(i);
    String maxKey = "flow_max_" + String(i);
    
    preferences.putFloat(factorKey.c_str(), flowCalibrationFactors[i]);
    preferences.putFloat(minKey.c_str(), flowMinLimits[i]);
    preferences.putFloat(maxKey.c_str(), flowMaxLimits[i]);
  }
  
  // Save pressure configurations
  for (int i = 0; i < 3; i++) {
    String minKey = "press_min_" + String(i);
    String maxKey = "press_max_" + String(i);
    
    preferences.putFloat(minKey.c_str(), pressureMinLimits[i]);
    preferences.putFloat(maxKey.c_str(), pressureMaxLimits[i]);
  }
}

void handleRoot() {
  String html = R"(
    <!DOCTYPE html>
    <html>
    <head>
        <title>Water Monitor - ESP32</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; background: #f0f2f5; }
            .container { max-width: 600px; margin: 0 auto; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
            .status { margin: 10px 0; padding: 10px; border-radius: 5px; }
            .connected { background: #d4edda; color: #155724; }
            .info { background: #d1ecf1; color: #0c5460; }
            h1 { color: #2196F3; text-align: center; }
            p { text-align: center; color: #666; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>💧 ESP32 Water Monitor</h1>
            <p>Multi-sensor water monitoring system is running</p>
            
            <div class="status connected">
                ✅ ESP32 Status: Online and Ready
            </div>
            
            <div class="status info">
                📡 Access the full monitoring interface by opening the separate HTML file and connecting to this IP address
            </div>
            
            <div class="status info">
                🔗 API Endpoints Available:
                <br>• /data - Get sensor readings
                <br>• /config - Get/Set sensor configuration
            </div>
        </div>
    </body>
    </html>
  )";
  server.send(200, "text/html", html);
}

void handleData() {
  // Create JSON response with sensor data
  DynamicJsonDocument doc(1000);
  
  // Flow meter data
  JsonArray flowRateArray = doc.createNestedArray("flowRates");
  JsonArray frequencyArray = doc.createNestedArray("frequencies");
  JsonArray pulseCountArray = doc.createNestedArray("pulseCounts");
  
  for (int i = 0; i < 3; i++) {
    flowRateArray.add(flowRates[i]);
    frequencyArray.add(frequencies[i]);
    pulseCountArray.add(pulseCounts[i]);
  }
  
  // Pressure data
  JsonArray pressureArray = doc.createNestedArray("pressures");
  JsonArray voltageArray = doc.createNestedArray("pressureVoltages");
  
  for (int i = 0; i < 3; i++) {
    pressureArray.add(pressures[i]);
    voltageArray.add(pressureVoltages[i]);
  }
  
  doc["timestamp"] = millis();
  
  String response;
  serializeJson(doc, response);
  
  server.send(200, "application/json", response);
}

void handleGetConfig() {
  DynamicJsonDocument doc(1000);
  
  // Flow meter configurations
  JsonArray flowFactors = doc.createNestedArray("flowFactors");
  JsonArray flowMins = doc.createNestedArray("flowMins");
  JsonArray flowMaxs = doc.createNestedArray("flowMaxs");
  
  for (int i = 0; i < 3; i++) {
    flowFactors.add(flowCalibrationFactors[i]);
    flowMins.add(flowMinLimits[i]);
    flowMaxs.add(flowMaxLimits[i]);
  }
  
  // Pressure configurations
  JsonArray pressMins = doc.createNestedArray("pressureMins");
  JsonArray pressMaxs = doc.createNestedArray("pressureMaxs");
  
  for (int i = 0; i < 3; i++) {
    pressMins.add(pressureMinLimits[i]);
    pressMaxs.add(pressureMaxLimits[i]);
  }
  
  String response;
  serializeJson(doc, response);
  
  server.send(200, "application/json", response);
}

void handleSetConfig() {
  if (server.hasArg("plain")) {
    DynamicJsonDocument doc(1000);
    deserializeJson(doc, server.arg("plain"));
    
    // Update flow meter configurations
    if (doc.containsKey("flowFactors")) {
      for (int i = 0; i < 3; i++) {
        flowCalibrationFactors[i] = doc["flowFactors"][i];
      }
    }
    if (doc.containsKey("flowMins")) {
      for (int i = 0; i < 3; i++) {
        flowMinLimits[i] = doc["flowMins"][i];
      }
    }
    if (doc.containsKey("flowMaxs")) {
      for (int i = 0; i < 3; i++) {
        flowMaxLimits[i] = doc["flowMaxs"][i];
      }
    }
    
    // Update pressure configurations
    if (doc.containsKey("pressureMins")) {
      for (int i = 0; i < 3; i++) {
        pressureMinLimits[i] = doc["pressureMins"][i];
      }
    }
    if (doc.containsKey("pressureMaxs")) {
      for (int i = 0; i < 3; i++) {
        pressureMaxLimits[i] = doc["pressureMaxs"][i];
      }
    }
    
    // Save configuration to flash
    saveConfiguration();
    
    server.send(200, "application/json", "{\"status\":\"success\"}");
  } else {
    server.send(400, "application/json", "{\"status\":\"error\",\"message\":\"No data received\"}");
  }
}
