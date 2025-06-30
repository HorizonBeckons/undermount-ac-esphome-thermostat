# Undermount AC V2 Hardware Adaptation

This repository contains an adapted version of the Undermount AC ESPHome Thermostat, specifically optimized for **Undermount AC V2 hardware systems**. The original V3 codebase had compatibility issues with V2 hardware, particularly preventing the fan from fully shutting off when the system was in OFF mode.

## ⚠️ Disclaimer

The software is provided "as is," without warranty of any kind, express or implied, including but not limited to warranties of merchantability, fitness for a particular purpose, and non-infringement. In no event shall the author be liable for any claim, damages, or other liability arising from the use of the software. This project is not associated with or endorsed by Undermount AC.

## Problem Statement

The primary issue addressed was documented in **GitHub Issue #18**, where the V3 code would not allow V2 systems to properly disable the fan. When the thermostat was set to OFF mode, the fan would continue operating at a low speed instead of shutting down completely.

## Hardware Differences: V2 vs V3

### V2 Connector Matrix (This Implementation)
- **Output 1** - Blower Speed (Red Wire)
- **Output 2** - Blower Power On/Off (Yellow Wire)
- **Output 3** - Compressor Power (Blue Wire)
- **Output 4** - High-Speed Compressor (Orange Wire)
- **Output 5** - Unused
- **Output 6** - Unused
- **POS** - DC+ IN (Red/White Wire)
- **NEG** - DC- IN (Black Wire)

### V3 Connector Matrix (Original)
- **Output 1** - Blower Speed (White Wire)
- **Output 2** - Blower Power On/Off (Brown Wire)
- **Output 3** - Compressor Power (Blue Wire)
- **Output 4** - High-Speed Compressor (Green Wire)
- **Output 5** - Unused
- **Output 6** - Unused

## Key Changes and Enhancements

### 🔧 True Fan OFF Capability
- **Problem**: V3 code maintained minimum 15% blower speed even in OFF mode
- **Solution**: Modified `blower_pwm_filter` to support true 0% operation
- **Implementation**: 
  ```yaml
  min_power: 0.00  # Changed from 0.15 to allow true off capability
  ```

### 🛡️ Enhanced Compressor Protection System
- Added comprehensive short-cycle protection with safety timers
- Implemented pending request handling for delayed shutdowns
- Added emergency safety checks to prevent compressor operation without blower

### 🎯 Improved PID Control
- Increased proportional gain from 0.2 to 0.5 for better responsiveness
- Enhanced PID logic with mode-specific minimum handling
- Added comprehensive debug logging for troubleshooting

### 🔒 Safety Enhancements
- **Evaporator Protection**: Enforced minimum 40% blower speed during cooling
- **High-Speed Protection**: Automatic 70% minimum when high-speed compressor active
- **Emergency Monitoring**: System monitor to detect unsafe conditions

### 🌪️ Fan Control Improvements
- **Fan-Only Mode**: Removed artificial minimums for natural PID control
- **Ramping Logic**: Enhanced to support true 0% shutdown
- **Mode Transitions**: Improved handling between cooling, idle, and off modes

## Critical Code Changes

### Blower PWM Filter Enhancement
```yaml
write_action:
  - lambda: |-
      // Handle true zero case
      if (state <= 0.0) {
        id(blower_pwm).set_level(0.0);
        id(blower_speed_called).publish_state(0.0);
        ESP_LOGD("Blower PWM Filter", "Blower OFF - State: %.2f, Duty: 0.00", state);
        return;
      }
```

### Compressor Protection Logic
```yaml
- lambda: |-
    // Check if short cycle protection should prevent this
    uint32_t time_since_start = millis() - id(compressor_start_time);
    bool protection_active = (time_since_start < 120000 && id(compressor_short_cycle_protection));
    
    if (protection_active) {
      ESP_LOGW("Protection", "Compressor turn-off blocked by short cycle protection");
      return; // Block the shutdown without forcing back on
    }
```

### Enhanced OFF Mode Handling
```yaml
off_mode:
  - switch.turn_off: compressor_power
  - switch.turn_off: compressor_speed_high
  - if:
      condition:
        lambda: return !id(compressor_power).state;
      then:
        # Normal shutdown sequence with evaporator purge
      else:
        # Maintain cooling airflow if compressor protection active
```

## New Diagnostic Features

### Protection Monitoring
- **Compressor Protection Time Remaining** - Shows active protection countdown
- **Compressor Short Cycle Protection Active** - Binary status indicator
- **PID Target Speed / PID Current Speed** - Real-time PID monitoring

### Safety Buttons
- **Trigger PID Test** - Manual PID execution for troubleshooting
- **Reset Cooling Minimum to Safe Default** - Emergency safety reset

## Installation and Usage

### Prerequisites
- Undermount AC V2 hardware system
- ESPHome environment
- Home Assistant (recommended)

### Configuration Steps
1. Use the provided `undermount-ac.yaml` configuration
2. Update WiFi credentials in the file
3. Flash to ESPHome Thermostat Controller 12-31V DC device
4. Connect to Undermount AC V2 hardware using the V2 connector matrix
5. Monitor diagnostic entities in Home Assistant for proper operation

## Safety Considerations

⚠️ **Never operate compressor without blower** - System includes automatic protection  
⚠️ **Minimum cooling speeds enforced** - 40% minimum during cooling to prevent evaporator freeze  
⚠️ **Short-cycle protection** - 2-minute minimum run/off times for compressor longevity  

## Testing and Validation

This adaptation has been thoroughly tested on V2 hardware with the following validation:

- ✅ Complete fan shutdown in OFF mode
- ✅ Proper compressor protection timing
- ✅ Evaporator freeze prevention
- ✅ Smooth PID control operation
- ✅ Emergency safety system activation

## Troubleshooting

### Fan Won't Turn Off
- Check diagnostic sensor **Output 2 (Blower Power)**
- Verify `min_power: 0.00` in `blower_pwm_filter` configuration
- Monitor **Ramp Blower** status for active ramping

### Compressor Issues
- Monitor **Compressor Protection Time Remaining**
- Check **Compressor Short Cycle Protection Active** status
- Review ESPHome logs for protection messages

### PID Performance
- Use **Trigger PID Test** button for manual testing
- Monitor **PID Target Speed** vs **PID Current Speed** sensors
- Adjust Kp value if response is too slow/fast

## Credits

- **Original Code**: anthonysecco - [Undermount AC V3 ESPHome Thermostat](https://github.com/anthonysecco/undermount-ac-esphome-thermostat)
- **Base Framework**: Mike Goubeaux (Smarty Van) - Foundation code
- **V2 Adaptation**: Chris Herman (HorizonBeckons) - Hardware compatibility and safety enhancements

## Contributing

When contributing to this V2-specific adaptation:

- Ensure all changes maintain V2 hardware compatibility
- Preserve safety protections and minimum speed requirements
- Test thoroughly on actual V2 hardware before submitting
- Document any new diagnostic features or safety mechanisms

## License

This project maintains the same license as the original Undermount AC ESPHome Thermostat repository.

---

**Note**: This adaptation specifically addresses V2 hardware compatibility while enhancing safety and diagnostic capabilities. For V3 hardware, please use the [original repository](https://github.com/anthonysecco/undermount-ac-esphome-thermostat).
