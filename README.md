# Undermount AC V2 ESPHome Thermostat with PID Control

An adapted version of the Undermount AC ESPHome Thermostat, specifically optimized for Undermount AC V2 hardware systems with comprehensive safety features and enhanced control logic.

## Disclaimer

The software is provided "as is," without warranty of any kind, express or implied, including but not limited to warranties of merchantability, fitness for a particular purpose, and non-infringement. In no event shall the author be liable for any claim, damages, or other liability arising from the use of the software. This project is not associated with or endorsed by Undermount AC.

## Overview

This repository contains an extensively modified version of the original V3 thermostat code, addressing multiple compatibility and operational issues with V2 hardware:

### Original Issues Addressed
- **Fan Shutdown Problem**: V3 code prevented V2 systems from fully shutting off the fan (GitHub Issue [#18](https://github.com/anthonysecco/undermount-ac-esphome-thermostat/issues/18))
- **Speed Control**: Fan speed control used incorrect fractional values instead of integer speeds
- **Compressor Restart**: System failed to restart compressor after reaching target temperature
- **State Management**: Poor handling of transitions between cooling/idle/off modes

### Version 0.4.5 Major Improvements
- ✅ Fixed blower fan speed control (now uses integer speeds 1-100)
- ✅ Enhanced cooling request handling with pending state management
- ✅ Improved compressor restart protection with automatic recovery
- ✅ Better state transitions between cooling/idle/off modes
- ✅ Fixed compressor restart after reaching target temperature
- ✅ Comprehensive safety features and protection systems
- ✅ Auto Mode High Speed Compressor - Tested and functional

## Hardware Compatibility

### V2 Connector Matrix (GPIO Pin Assignments)
```
GPIO 7  - Output 1 - Blower Speed (Red Wire)
GPIO 9  - Output 2 - Blower Power On/Off (Yellow Wire)
GPIO 8  - Output 3 - Compressor Power (Blue Wire)
GPIO 10 - Output 4 - High-Speed Compressor (Orange Wire)
Output 5 - Unused
Output 6 - Unused
POS - DC+ IN (Red/White Wire)
NEG - DC- IN (Black Wire)
```

### V3 Wire Colors (for reference)
```
Output 1 - White Wire
Output 2 - Brown Wire
Output 3 - Blue Wire (same as V2)
Output 4 - Green Wire
```

## Key Features

### Safety Systems
1. **Compressor Protection**
   - 2-minute minimum run time (prevents short cycling)
   - 2-minute restart delay after shutdown
   - Automatic recovery when protection expires
   - Manual safety override for emergencies

2. **Emergency Monitoring**
   - Detects compressor running without blower
   - Automatic emergency blower activation
   - Safety status reporting via diagnostic sensors

3. **Intelligent State Management**
   - Pending state system for delayed transitions
   - Proper handling of rapid temperature changes
   - Automatic compressor restart when protection allows

### Control Features
1. **Enhanced PID Control**
   - Kp = 0.2, Ki = 0.000334, Kd = 0.0
   - Dynamic integral clamping (factor: 1.5)
   - Mode-specific minimum speeds
   - 10-second update interval in AUTO mode

2. **Fan Speed Management**
   - Integer speed control (1-100) for proper operation
   - 10-second ramping for smooth transitions
   - True 0% shutdown capability
   - Mode-specific minimums:
     - Cooling: 40-60% (configurable)
     - Fan-only: 20%
     - Off/Idle: 0%

3. **Operating Modes**
   - **OFF**: Complete shutdown with proper sequencing
   - **COOL**: Full cooling with compressor and fan control
   - **FAN_ONLY**: Fan operation without compressor
   - **AUTO Fan Mode**: PID-controlled fan speeds
   - **Manual Fan Modes**: LOW (25%), MEDIUM (50%), HIGH (100%)

4. **High-Speed Compressor Control** (Tested & Functional)
   - Automatic activation when blower speed ≥75% for 2 minutes
   - Automatic deactivation when speed <75% for 2 minutes
   - Manual test mode available via switch
   - Enable/disable auto mode via configuration switch

## Technical Implementation

### Blower PWM Filter
```yaml
min_power: 0.00  # Allows true shutdown
max_power: 0.97  # Maximum 97% as per hardware specs
write_action:
  # Maps 0-100% input to appropriate PWM duty cycle
  # 0% = off, 1-20% = 20% minimum, 20-97% = linear mapping
```

### Compressor Protection Logic
```yaml
# Pending states:
# 0 = none
# 1 = idle pending
# 2 = off pending  
# 3 = cooling pending (NEW - for restart after protection)
```

### Fan Speed Control Fix
```yaml
fan:
  - platform: speed
    speed_count: 100  # Critical for integer speed control
    # Fan now expects speed values 1-100, not 0.01-1.0
```

## Diagnostic Entities

### Protection Monitoring
- **Compressor Protection Time Remaining**: Countdown timer showing seconds until protection expires
- **Compressor Protection Active**: Binary indicator of active protection
- **High-Speed Status**: Shows if high-speed compressor mode is active

### System Status
- **Output 1 (Blower Speed)**: Current PWM percentage
- **Output 2 (Blower Power)**: Blower power supply status
- **Output 3 (Compressor Power)**: Compressor on/off status
- **Output 4 (High-Speed)**: High-speed compressor status

### Control Switches
- **Enable High-speed Compressor**: Auto high-speed mode toggle
- **High-Speed Test Switch**: Manual high-speed testing
- **Compressor Safety Override**: Emergency protection bypass

## Installation

1. **Prerequisites**
   - Undermount AC V2 hardware system
   - ESPHome environment configured
   - Home Assistant (recommended for full functionality)

2. **Configuration**
   - Use the provided `undermount-ac.yaml` configuration
   - Update WiFi credentials and API settings
   - Adjust PID parameters if needed for your system

3. **Flashing**
   - Compile in ESPHome
   - Flash to ESPHome Thermostat Controller (12-31V DC)
   - Connect using V2 wire color scheme

4. **Initial Testing**
   - Verify all diagnostic entities appear in Home Assistant
   - Test each mode (OFF, COOL, FAN_ONLY)
   - Monitor protection timers during operation
   - Verify fan properly shuts off in OFF mode
   - Test high-speed compressor activation (manual and auto)

## Testing & Validation

This adaptation has been thoroughly tested on V2 hardware with full validation of all features (as of v0.4.5):

### Confirmed Working
- ✅ Complete fan shutdown in OFF mode
- ✅ Proper compressor protection timing
- ✅ Evaporator freeze prevention
- ✅ Smooth PID control operation
- ✅ Emergency safety system activation
- ✅ Integer fan speed control (1-100)
- ✅ Compressor restart after temperature changes
- ✅ High-speed compressor automatic control
- ✅ All state transitions (OFF/IDLE/COOL/FAN_ONLY)
- ✅ Manual fan speed modes (LOW/MEDIUM/HIGH)
- ✅ Safety override functionality

### High-Speed Compressor Operation
- Activates automatically when fan speed ≥75% for 2 minutes continuous
- Deactivates when fan speed drops <75% for 2 minutes
- Manual test mode allows immediate activation for testing
- Can be disabled via configuration switch if not desired

## Troubleshooting

### Fan Won't Turn Off
1. Check `min_power: 0.00` in blower_pwm_filter
2. Verify fan speed shows 0% in diagnostics
3. Check for emergency safety activation in logs

### Compressor Won't Start
1. Monitor "Compressor Protection Time Remaining"
2. Check if protection is active (2-minute timer)
3. Use safety override switch if needed (emergency only)

### Fan Speed Issues
1. Ensure `speed_count: 100` is set in fan configuration
2. Verify speed values are integers (1-100)
3. Check ramping status in debug logs

### Temperature Control Problems
1. Adjust PID parameters:
   - Increase Kp for faster response
   - Increase Ki for better steady-state
2. Check temperature sensor readings
3. Verify proper mode selection

## Safety Warnings

⚠️ **NEVER** disable compressor protection unless absolutely necessary
⚠️ **ALWAYS** monitor system after using safety override
⚠️ Ensure proper ventilation around the AC unit

## Version History

### v0.4.5 (Current)
- All features tested and validated on V2 hardware
- High-speed compressor functionality confirmed working
- Documentation updated to reflect complete testing status

### v0.4.4
- Fixed integer fan speed control
- Added pending cooling state for restart management
- Enhanced state transition logic
- Improved compressor protection handling
- Better diagnostic reporting

### v0.4.0
- Initial V2 hardware adaptation
- Basic compressor protection implementation
- PID control enhancements

## Credits

- **Original V3 Code**: anthonysecco - [Undermount AC V3 ESPHome Thermostat](https://github.com/anthonysecco/undermount-ac-esphome-thermostat)
- **Base Framework**: Mike Goubeaux (Smarty Van)
- **V2 Adaptation**: Chris Herman (HorizonBeckons)
- **Contributors**: Thanks to all who reported issues and tested fixes

## Contributing

When contributing to this V2-specific adaptation:
1. Test all changes on actual V2 hardware
2. Maintain backward compatibility with existing configurations
3. Preserve all safety features
4. Document any new features or changes
5. Submit detailed pull requests with test results

## License

This project maintains the same license as the original Undermount AC ESPHome Thermostat repository.

## Support

For V2-specific issues, please use the [Issues](https://github.com/HorizonBeckons/undermount-ac-esphome-thermostat/issues) section of this repository.

For V3 hardware, please use the [original repository](https://github.com/anthonysecco/undermount-ac-esphome-thermostat).

---

**Note**: This adaptation specifically addresses V2 hardware compatibility while enhancing safety and diagnostic capabilities. Always ensure your system is properly configured before operation.