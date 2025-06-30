# Undermount AC V2 System - Enhanced Controller

## Overview
This is an enhanced version of the [original Undermount AC controller](https://github.com/anthonysecco/undermount-ac-esphome-thermostat), specifically adapted for **v2 hardware** with additional safety and control improvements.

## Credits
- **Original Project:** [anthonysecco/undermount-ac-esphome-thermostat](https://github.com/anthonysecco/undermount-ac-esphome-thermostat) for Undermount AC v3
- **Originally written by:** [anthonysecco](https://github.com/anthonysecco) 
- **Special thanks to:** Mike Goubeaux (Smarty Van) for the base code
- **v2 Adaptation and Enhancements by:** Chris Herman [HorizonBeckons]

## Hardware Compatibility
⚠️ **Important:** This fork is specifically designed for **Undermount AC v2 systems**. 

### V2 Undermount AC System Harness to ESPHome HVAC Controller Connector Matrix:
- **Output 1** - Blower Speed (Red Wire)
- **Output 2** - Blower Power On/Off (Yellow Wire) 
- **Output 3** - Compressor Power (Blue Wire)
- **Output 4** - High-Speed Compressor Wire
- **Output 5** - Unused
- **Output 6** - Unused
- **POS** - DC+ IN (Red/White Wire)
- **NEG** - DC- IN (Black Wire)

### Key Differences from Original v3 Code:
- **Blower Speed:** PWM 50-500Hz Compatible (Version 3 uses 240Hz, also tested at 50Hz on v2)
- **Wire Colors:** Updated connector matrix for v2 hardware (Red vs White for blower speed, Yellow vs Brown for blower power)
- Ported and tested for v2 hardware compatibility
- Validated on v2 hardware with cursory testing
- All safety features tested and confirmed working on v2 system **except Auto High Speed Compressor Mode**

## My Additions to the Original Project

*All features below have been enhanced or added for V2 system compatibility. Based on the excellent foundation provided by the original project.*

### 🛡️ Compressor Protection System 
- **Short-cycle protection:** Prevents compressor damage from rapid on/off cycling
- **Timing safeguards:** 2-minute minimum run/off times with smart override logic
- **Delayed shutdown scripts:** Proper evaporator purging sequences
- **Protection status monitoring:** Real-time display of remaining protection time

### 🎯 Enhanced PID Control 
- **Multiple trigger intervals:** Separate PID loops for cooling vs fan-only modes (2s for cooling, 5s for fan-only)
- **Force fan-only start:** Immediate fan startup script to overcome thermostat delays
- **Improved responsiveness:** Additional PID triggers for better temperature control

### 📊 Advanced Diagnostics 
- **Protection timers:** Real-time sensors showing remaining protection time
- **System state monitoring:** Additional binary sensors for compressor, fan, and protection status
- **Emergency detection:** 10-second interval checking for dangerous conditions
- **Enhanced sensor filtering:** Additional temperature stability filters

### 🚨 Safety Features 
- **Evaporator protection:** Enhanced minimum airflow logic with configurable settings (40-60%)
- **Emergency fan restart:** Automatic recovery if fan stops during compressor operation
- **Smart state management:** Advanced handling of protection conflicts and pending shutdown requests
- **High-speed compressor protection:** 70% minimum airflow enforcement when high-speed active

### 🔧 Control Improvements 
- **True zero capability:** Modified PWM filter for complete fan shutdown (min_power: 0.00 vs 0.15)
- **Enhanced logging:** Changed from INFO to VERY_VERBOSE mode for better debugging
- **Improved PWM filtering:** Enhanced zero-state handling and hysteresis logic
- **Climate timing adjustments:** Reduced minimum times for better fan-only responsiveness

## Installation
1. Flash this YAML to your Undermount AC ESPHome HVAC ESP32-S3 controller
2. Configure WiFi credentials in the `wifi:` section
3. Set your minimum cooling blower speed in Home Assistant
4. Enabling high-speed compressor allows for the auto functionality, it does not trigger high-speed.

## Configuration
Key settings you can adjust:
- **Minimum Cooling Blower Speed:** 40-60% (configurable in Home Assistant)
- **PID Parameters:** Already tuned, but adjustable in substitutions
- **Protection Timers:** 2-minute defaults (HVAC system requirement)

## Testing
This code has been tested on v2 hardware including:
- ✅ Normal cooling operation
- ✅ Fan-only mode  
- ✅ Protection system activation
- ✅ Emergency scenarios and state transitions
- ⚠️ **Note:** Auto High Speed Compressor Mode not fully tested on v2 hardware
- ✅ All other safety features confirmed working

## Technical Details
- **Platform:** ESP32-S3 with ESPHome
- **Sensors:** SHT30 temperature/humidity
- **Outputs:** 6 channels (4 used for AC control)
- **Control:** Advanced PID with integral windup protection

## Contributing
Feel free to fork and improve! This is open source hardware control at its best.

## License
Maintains original open source licensing. See original repository for details.
