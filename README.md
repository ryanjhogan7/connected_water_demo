# Multi-Sensor Water Monitoring System

A comprehensive IoT water monitoring solution using ESP32 microcontroller with real-time web interface for R&D and commercial water applications.

## Features

- **Multi-sensor support**: 3 pressure transducers + 3 flow meters
- **Real-time monitoring**: Live data streaming via WiFi
- **Web-based interface**: Responsive HTML dashboard
- **Configurable sensors**: Adjustable calibration and range settings
- **Data logging**: CSV export with customizable intervals
- **CORS-resilient**: Robust frontend handling of cross-origin requests

## Hardware Requirements

### ESP32 Development Board
- **Recommended**: ESP32-C6-DevKitC-1 or ESP32-DEVKITC
- **Power Supply**: USB or external 5V supply (ESP32 5V pin powers all sensors)
- **Signal Conditioning Components**: 
  - **3x 165Ω precision resistors** (1% tolerance) for pressure current shunts
  - **3x 2.2kΩ resistors** for flow meter voltage dividers
  - **3x 1.5kΩ resistors** for flow meter voltage dividers
  - **Breadboard or PCB** for signal conditioning circuits
  - **Jumper wires** for connections

### Sensors
- **Pressure Transducers**: 3x units with 4-20mA current output (5V supply from ESP32)
- **Flow Meters**: 3x pulse-output flow meters with 5V logic levels (5V supply from ESP32)
  - Small flow meters: F = 21*Q (0.3-10.0 L/min range)
  - Big flow meters: F = 7.5*Q (1.0-50.0 L/min range)

### Pin Configuration (ESP32-C6)
```
Pressure Sensors (current measurement via shunts):
- Pressure 1: GPIO 0 (ADC1_CH0) + shunt resistor
- Pressure 2: GPIO 1 (ADC1_CH1) + shunt resistor
- Pressure 3: GPIO 2 (ADC1_CH2) + shunt resistor

Flow Meters (voltage level conversion):
- Flow 1: GPIO 18 (Digital Input) + voltage divider
- Flow 2: GPIO 19 (Digital Input) + voltage divider
- Flow 3: GPIO 20 (Digital Input) + voltage divider
```

### Wiring
```
Flow Meter Connections (powered by ESP32):
- Red (VCC+) → ESP32 5V Pin
- Black (GND-) → ESP32 GND
- Yellow (Signal) → Voltage Divider → ESP32 Digital Pin

Pressure Transducer Connections (powered by ESP32):
- VCC+ → ESP32 5V Pin
- Signal+ → Shunt Resistor → ESP32 ADC Pin
- Signal- & GND → ESP32 GND
```

### Signal Conditioning Circuits

#### For Flow Meters (5V Pulse → 3.3V Logic)
**3x Voltage Dividers Required:**

```
5V Flow Signal ─── 2.2kΩ ─── ESP32 Digital Pin
                       │
                     1.5kΩ
                       │
                     GND
```

#### For Pressure Sensors (4-20mA → 0-3.3V)
**3x Current-to-Voltage Shunts Required:**

```
Pressure Sensor (+) ─── 165Ω Shunt ─── ESP32 ADC Pin
                             │
Pressure Sensor (-) ─────────┴─────── ESP32 GND
```

**Current-to-Voltage Conversion:**
- **4mA × 165Ω = 0.66V** (minimum pressure)
- **20mA × 165Ω = 3.30V** (maximum pressure)
- **Perfect 0.66V-3.30V range** for ESP32 ADC

### Power Considerations
**Important**: ESP32 5V pin current capacity varies by board:
- **ESP32-DevKitC**: ~500mA available from USB
- **Total sensor current**: Calculate actual sensor consumption
- **Recommendation**: Use external 5V supply if sensors draw >300mA combined
- **Alternative**: Power ESP32 via VIN pin with 7-12V supply for higher 5V current capacity

## Software Setup

### Arduino IDE Requirements
- **ESP32 Board Package**: Install via Board Manager
- **Libraries**:
  - `WiFi.h` (built-in)
  - `WebServer.h` (built-in)
  - `ArduinoJson` (install via Library Manager)
  - `Preferences.h` (built-in)

### Installation Steps

1. **Clone/Download** the project files
2. **Open** `esp32_water_monitor.ino` in Arduino IDE
3. **Configure WiFi credentials**:
   ```cpp
   const char* WIFI_SSID = "YOUR_WIFI_NAME";
   const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
   ```
4. **Upload** the code to your ESP32
5. **Note the IP address** from Serial Monitor
6. **Open** `water_monitor_interface.html` in web browser
7. **Connect** using the ESP32's IP address

## System Architecture

```
┌─────────────────┐    WiFi    ┌─────────────────┐
│   Web Browser   │◄──────────►│     ESP32       │
│   (Dashboard)   │   HTTP     │  (Web Server)   │
└─────────────────┘            └─────────────────┘
                                        │
                               ┌────────┴────────┐
                               │                 │
                         ┌─────▼─────┐    ┌─────▼─────┐
                         │ Pressure  │    │   Flow    │
                         │ Sensors   │    │  Meters   │
                         │  (3x)     │    │   (3x)    │
                         └───────────┘    └───────────┘
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | System status page |
| `/data` | GET | Current sensor readings (JSON) |
| `/config` | GET | Get sensor configuration |
| `/config` | POST | Save sensor configuration |

### Sample JSON Response (`/data`)
```json
{
  "flowRates": [1.23, 0.45, 2.10],
  "frequencies": [25, 9, 44], 
  "pulseCounts": [25, 9, 44],
  "pressures": [15.2, 8.7, 22.1],
  "pressureVoltages": [1.234, 0.987, 1.876],
  "timestamp": 123456789
}
```

## Web Interface Features

### Dashboard Sections
- **Sensor View**: Real-time data display with visual cards
- **Configuration**: Adjust sensor parameters and calibration
- **Data Logging**: Record and export data with customizable intervals

### Configuration Options
- **Flow Meter Types**: Small (F=21*Q) or Big (F=7.5*Q)
- **Flow Ranges**: Customizable min/max values (L/min)
- **Pressure Ranges**: Adjustable PSI ranges for each sensor
- **Data Intervals**: 0.5-600 second logging intervals

### Data Export
- **CSV Format**: Timestamped data with all sensor readings
- **Automatic Naming**: Files include date, time, and interval info
- **Comprehensive Data**: Flow rates, pressures, voltages, and metadata

## Troubleshooting

### Connection Issues
1. **Check WiFi credentials** in ESP32 code
2. **Verify IP address** from Serial Monitor
3. **Ensure same network** for ESP32 and computer
4. **Try "Test Endpoints"** button for diagnostics

### CORS Errors
- The system includes CORS-resilient code
- Configuration may show warnings but should still work
- Check browser console (F12) for detailed error info

### Sensor Calibration
1. **Pressure sensors**: Use known pressure source; verify 4-20mA output with multimeter
2. **Flow meters**: Verify pulse output with known flow rate  
3. **Shunt resistors**: Verify 165Ω ±1% resistance for accurate current measurement
4. **Voltage dividers**: Check 5V flow signals give ~2.03V at ESP32 pins
5. **Current loop verification**: 4mA should read ~0.66V, 20mA should read ~3.30V at ADC
6. **Digital flow signals**: Ensure clean 0V/3.3V logic levels after voltage division
7. **Power verification**: Check ESP32 5V pin provides stable voltage under sensor load

### Common Issues
| Problem | Solution |
|---------|----------|
| No data updates | Check ESP32 power and WiFi connection |
| Incorrect pressure readings | Verify 4-20mA current loop and 165Ω shunt resistor values |
| Flow meters not counting | Check voltage dividers on flow signal lines |
| Pressure readings stuck at min/max | Check current loop wiring and ESP32 5V power supply |
| Sensors not responding | Check ESP32 5V pin current capacity vs sensor requirements |
| Config won't save | Try browser refresh or incognito mode |
| ADC readings outside 0.66V-3.30V | Check pressure sensor current loop and shunt resistor |
| Intermittent flow readings | Ensure voltage dividers provide clean 0V/3.3V switching |
| Current loop noise | Use twisted pair wiring for pressure sensor connections |
| ESP32 resets under load | Sensors drawing too much current; use external 5V supply |

## Technical Specifications

### Performance
- **Update Rate**: Real-time (500ms display updates)
- **Data Logging**: User-configurable intervals (0.5-600s)
- **Memory**: Stores up to 1000 data points in browser
- **Precision**: Flow (0.01 L/min), Pressure (0.1 PSI)

### Limitations
- **ESP32 ADC**: 12-bit resolution (0-4095 counts)
- **Current loop resolution**: Limited by 165Ω shunt precision and ADC resolution
- **Flow frequency**: Limited by ESP32 interrupt handling and voltage divider response
- **Power capacity**: ESP32 5V pin typically limited to 500mA; check sensor current draw
- **Current loop distance**: 4-20mA signals good for longer cable runs (advantage)
- **Network**: Requires 2.4GHz WiFi (ESP32 limitation)

## Development Notes

### Code Structure
- **Interrupt-driven**: Flow meters use hardware interrupts
- **Non-blocking**: Web server handles multiple clients
- **Flash storage**: Configuration saved to ESP32 flash memory
- **JSON communication**: RESTful API with ArduinoJson library

### Calibration Formula
```cpp
// Flow rate calculation (accounting for voltage divider)
// Digital pulse counting unaffected by voltage scaling
flowRate = frequency / calibrationFactor;

// Pressure calculation (4-20mA current loop with shunt)
// Shunt converts: 4mA=0.66V (min), 20mA=3.30V (max)
currentmA = (adcVoltage - 0.66) / (3.30 - 0.66) * 16 + 4; // Convert to 4-20mA
normalizedCurrent = (currentmA - 4.0) / 16.0; // Normalize to 0.0-1.0
pressure = minPSI + (normalizedCurrent * (maxPSI - minPSI));
```

## Future Enhancements

- [ ] Add temperature sensor support
- [ ] Implement TDS (Total Dissolved Solids) monitoring
- [ ] Add data visualization charts
- [ ] Include alarm/notification system
- [ ] Add MQTT integration for IoT platforms
- [ ] Implement user authentication
- [ ] Add mobile app compatibility

## License

This project is provided as-is for educational and research purposes. Modify and use according to your requirements.

## Support

For technical issues:
1. Check Serial Monitor output for ESP32 diagnostics
2. Use browser developer tools (F12) for frontend debugging
3. Verify hardware connections and power supply
4. Ensure all required libraries are installed

---

**Version**: 1.0  
**Last Updated**: January 2025  
**Compatible**: ESP32-C6, ESP32-DEVKITC  
**Requirements**: Arduino IDE, compatible sensors
