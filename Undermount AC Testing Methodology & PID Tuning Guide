# Undermount AC Testing Methodology & PID Tuning Guide

## Table of Contents
1. [Pre-Testing Setup](#pre-testing-setup)
2. [Safety System Testing](#safety-system-testing)
3. [Basic Functionality Testing](#basic-functionality-testing)
4. [PID Control Testing](#pid-control-testing)
5. [PID Tuning Methodology](#pid-tuning-methodology)
6. [Advanced Feature Testing](#advanced-feature-testing)
7. [Integration Testing](#integration-testing)
8. [Troubleshooting Guide](#troubleshooting-guide)

## Pre-Testing Setup

### Essential Equipment
- **Multimeter** for voltage/current measurements
- **Temperature gun or probe** for evaporator monitoring
- **Oscilloscope** (optional) for PWM signal analysis
- **Home Assistant instance** for monitoring
- **Safety equipment** (safety glasses, insulated tools)

### Initial Configuration
```yaml
# Enable all diagnostic sensors for testing
sensor:
  - blower_speed_called: disabled_by_default: false
  - compressor_protection_remaining: disabled_by_default: false
  - compressor_short_cycle_remaining: disabled_by_default: false
  - pid_target_speed: disabled_by_default: false
  - pid_current_speed: disabled_by_default: false

# Set conservative initial PID values
substitutions:
  kp: '0.3'                    # Start conservative
  ki: '0.000167'               # Half the default
  kd: '0.0'                    # Keep at 0 initially
  integral_clamp_factor: '1.0' # Start with 1.0
```

### Pre-Test Checklist
- [ ] All connections secure per V2 connector matrix
- [ ] DC power supply within specification (12V/24V as required)
- [ ] Home Assistant connectivity confirmed
- [ ] All diagnostic entities enabled
- [ ] Evaporator clean and unobstructed
- [ ] Ambient temperature measurement calibrated

## Safety System Testing

### Test 1: Compressor Protection System
**Objective**: Validate 2-minute compressor protection delays

**Steps**:
1. Set thermostat to COOL mode, target 5°F below current temp
2. Wait for compressor to start
3. Monitor `compressor_protection_remaining` sensor
4. Switch to OFF mode immediately
5. Verify compressor turns off and protection timer starts
6. Try to restart cooling before timer expires
7. Confirm compressor stays off until timer reaches 0

**Expected Results**:
- Timer shows 120 seconds and counts down
- Compressor restart blocked during protection period
- Blower may continue for evaporator protection during protection period

**Pass Criteria**: ✅ Protection timer functions, compressor restart properly delayed

### Test 2: Short Cycle Protection
**Objective**: Ensure compressor runs minimum 2 minutes before allowing shutdown

**Steps**:
1. Start cooling mode
2. Wait 30 seconds, then switch to OFF
3. Monitor `compressor_short_cycle_remaining` sensor
4. Verify compressor continues running
5. Verify blower maintains cooling airflow
6. Wait for protection to expire (120 seconds total runtime)
7. Confirm system shuts down properly after protection expires

**Expected Results**:
- Compressor continues running despite OFF command
- `pending_off_request` becomes true
- Blower maintains minimum cooling speed
- System executes delayed shutdown after 2 minutes

**Pass Criteria**: ✅ Short cycle protection prevents premature shutdown

### Test 3: Evaporator Freeze Protection
**Objective**: Validate minimum blower speeds during cooling

**Steps**:
1. Set `cooling_min_power` to 40%
2. Start cooling mode
3. Monitor blower speed during PID operation
4. Verify speed never drops below 40% while compressor runs
5. Test with high-speed compressor active
6. Verify 70% minimum enforced during high-speed operation

**Expected Results**:
- Blower speed ≥ 40% during normal cooling
- Blower speed ≥ 70% during high-speed cooling
- PID output properly clamped to minimums

**Pass Criteria**: ✅ Minimum speeds enforced, evaporator protected

### Test 4: Emergency Safety Check
**Objective**: Test compressor-without-blower detection

**Steps**:
1. Start cooling mode normally
2. Manually turn off blower fan entity in Home Assistant
3. Monitor system response within 5 seconds
4. Verify automatic blower restart
5. Check error logging

**Expected Results**:
- System detects condition within 5 seconds
- Blower automatically restarted
- Safety warning logged
- Proper minimum speed enforced

**Pass Criteria**: ✅ Emergency condition detected and corrected

## Basic Functionality Testing

### Test 5: Fan Mode Operations
**Objective**: Validate all fan modes work correctly

**Test each mode**:
- **AUTO**: PID should control speed based on temperature
- **LOW**: Fixed ~1% speed
- **MEDIUM**: Fixed 50% speed  
- **HIGH**: Fixed 100% speed
- **OFF**: Complete shutdown

**Steps for each mode**:
1. Set fan mode
2. Monitor `blower_speed_called` sensor
3. Verify target vs actual speed alignment
4. Test mode transitions
5. Verify ramp behavior

**Pass Criteria**: ✅ Each mode produces expected, stable speeds

### Test 6: Climate Mode Transitions
**Objective**: Test smooth transitions between climate modes

**Test Matrix**:
- OFF → COOL → OFF
- OFF → FAN_ONLY → OFF  
- COOL → FAN_ONLY → COOL
- COOL → IDLE → COOL

**Monitor during each transition**:
- Compressor state changes
- Blower speed changes
- Protection timer behavior
- Minimum speed enforcement

**Pass Criteria**: ✅ Smooth transitions, no unexpected states

### Test 7: Preset Functionality
**Objective**: Validate preset temperature and mode settings

**Test each preset**:
- **Standby**: 75°F, AUTO fan, OFF mode
- **Home**: 75°F, AUTO fan, COOL mode
- **Sleep**: 72°F, AUTO fan, COOL mode  
- **Away**: 90°F, AUTO fan, COOL mode

**Pass Criteria**: ✅ Presets apply correct settings, restore on boot

## PID Control Testing

### Test 8: PID Response Characterization
**Objective**: Understand system response before tuning

**Steps**:
1. Set conservative PID values (kp=0.3, ki=0.000167, kd=0.0)
2. Set target 5°F below current temperature
3. Start cooling mode with AUTO fan
4. Record data every 30 seconds for 30 minutes:
   - Current temperature
   - Target temperature  
   - Error (current - target)
   - PID target speed
   - PID current speed
   - Actual blower speed
   - Compressor status

**Data Collection Template**:
```
Time | Temp | Target | Error | PID_Target | PID_Current | Blower | Compressor
-----|------|--------|-------|------------|-------------|--------|------------
0:00 | 25.5 | 22.2   | 3.3   | 45         | 0           | 40     | ON
0:30 | 25.2 | 22.2   | 3.0   | 50         | 42          | 42     | ON
...
```

**Analysis Points**:
- Time to reach steady state
- Overshoot/undershoot behavior
- Oscillation frequency and amplitude
- Steady-state error

### Test 9: Step Response Test
**Objective**: Measure system response to step changes

**Steps**:
1. Stabilize at comfortable temperature
2. Make 3°F step change in setpoint
3. Record response for 20 minutes
4. Analyze rise time, settling time, overshoot

**Key Metrics**:
- **Rise Time**: Time to reach 90% of final value
- **Settling Time**: Time to stay within ±0.5°F of target
- **Overshoot**: Maximum temperature below target
- **Steady-State Error**: Final error after settling

## PID Tuning Methodology

### Understanding Current Settings
Your current PID values:
- **kp = 0.5**: Adds 50% blower speed per 1°C error
- **ki = 0.000334**: Integral action over ~10 minutes
- **kd = 0.0**: No derivative action (recommended for HVAC)

### Phase 1: Proportional Gain (Kp) Tuning
**Objective**: Find optimal proportional response

**Method**: Ziegler-Nichols inspired approach
1. Set ki = 0, kd = 0 (proportional only)
2. Start with kp = 0.1
3. Make step change in setpoint
4. Gradually increase kp until system oscillates
5. Record critical gain (Kc) where oscillation starts
6. Set kp = 0.6 × Kc as starting point

**Tuning Guidelines**:
```yaml
# Too Low (kp < 0.2): Slow response, large steady-state error
# Good Range (kp = 0.3-0.7): Quick response, minimal overshoot  
# Too High (kp > 1.0): Oscillations, instability
```

**Test each value for 20+ minutes**:
- kp = 0.2, ki = 0, kd = 0
- kp = 0.4, ki = 0, kd = 0  
- kp = 0.6, ki = 0, kd = 0
- kp = 0.8, ki = 0, kd = 0

### Phase 2: Integral Gain (Ki) Tuning
**Objective**: Eliminate steady-state error without windup

**Method**: Gradual increase approach
1. Use best kp from Phase 1
2. Start with ki = 0.0001
3. Increase gradually until steady-state error eliminated
4. Watch for integral windup or overshoot

**Tuning Guidelines**:
```yaml
# Ki Calculation: To reach 100% output in X minutes with 1°C error
# ki = 0.00167 for 10 minutes  
# ki = 0.00083 for 20 minutes
# ki = 0.00033 for 30 minutes (current setting)
```

**Test progression**:
- ki = 0.0001 (very slow)
- ki = 0.0003 (current value)  
- ki = 0.0005 (faster)
- ki = 0.0007 (aggressive)

### Phase 3: Integral Clamp Tuning
**Objective**: Prevent integral windup

**Current setting**: `integral_clamp_factor = 1.5`

**Test values**:
- 1.0: Conservative, less overshoot
- 1.5: Current, balanced approach
- 2.0: Aggressive, faster response

### Recommended Tuning Sequences

#### For Quick Cooling (Aggressive)
```yaml
kp: '0.7'
ki: '0.0005'  
kd: '0.0'
integral_clamp_factor: '2.0'
```

#### For Comfort (Balanced) - Recommended Starting Point
```yaml
kp: '0.5'
ki: '0.000334'
kd: '0.0' 
integral_clamp_factor: '1.5'
```

#### For Quiet Operation (Conservative)
```yaml
kp: '0.3'
ki: '0.0002'
kd: '0.0'
integral_clamp_factor: '1.0'
```

### Environmental Testing Scenarios

#### Scenario 1: High Heat Load (Summer Afternoon)
- Set target 10°F below ambient
- Monitor system response
- Verify high-speed compressor engagement
- Check for temperature overshoot

#### Scenario 2: Low Heat Load (Mild Conditions)  
- Set target 3°F below ambient
- Ensure system doesn't short cycle
- Verify smooth low-speed operation

#### Scenario 3: Rapid Load Changes
- Simulate door opening (temperature spike)
- Monitor recovery time
- Check for excessive blower speed changes

## Advanced Feature Testing

### Test 10: High-Speed Compressor Logic
**Objective**: Validate automated high-speed operation

**Prerequisites**: `enable_high_speed` switch ON

**Steps**:
1. Create high cooling demand (target 8°F+ below current)
2. Monitor blower speed progression
3. Verify high-speed starts at 75% blower speed
4. Confirm 2-minute delay protection
5. Test high-speed stop at 65% blower speed
6. Verify minimum speed enforcement (70%) during high-speed

**Expected Behavior**:
- High-speed engages after blower ≥75% for 2 minutes
- Minimum blower speed increases to 70% during high-speed
- High-speed disengages after blower ≤65% for 2 minutes

### Test 11: Fan-Only Mode PID
**Objective**: Validate PID control in fan-only mode

**Steps**:
1. Set mode to FAN_ONLY
2. Set target 2°F below current temperature
3. Monitor PID response without compressor
4. Verify no artificial minimums applied
5. Test circulation speed when near setpoint

**Expected Results**:
- PID operates normally without compressor
- No 40% minimum enforced (fan-only uses 0% minimum)
- Comfort circulation (25%) when error <0.5°C

### Test 12: Blower Ramp Control
**Objective**: Test smooth speed transitions

**Steps**:
1. Monitor during various speed changes
2. Verify 10-second ramp time (0-100%)
3. Test ramp interruption and direction changes
4. Confirm ramp-to-zero completely stops fan

**Pass Criteria**: ✅ Smooth transitions, no abrupt speed changes

## Integration Testing

### Test 13: Home Assistant Integration
**Objective**: Verify complete HA functionality

**Test Features**:
- [ ] Climate entity controls work
- [ ] All sensors report correctly
- [ ] Diagnostic entities provide useful data
- [ ] Switches function properly
- [ ] Preset changes apply correctly
- [ ] Status LED reflects system state

### Test 14: Long-Duration Stability
**Objective**: 24-hour continuous operation test

**Monitoring Points**:
- Memory usage stability
- Wi-Fi connectivity
- Temperature control accuracy
- Compressor cycle frequency
- Any error conditions

### Test 15: Edge Case Handling
**Objective**: Test system behavior under unusual conditions

**Test Cases**:
- Wi-Fi disconnection during operation
- Power interruption and recovery
- Sensor failures (simulate with bad readings)
- Extreme temperature setpoints
- Rapid mode changes

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue: PID Oscillations
**Symptoms**: Temperature bounces around setpoint
**Solutions**:
- Reduce kp by 20%
- Increase integral_clamp_factor
- Check for air leaks affecting temperature sensor

#### Issue: Slow Response
**Symptoms**: Takes too long to reach setpoint
**Solutions**:
- Increase kp gradually
- Increase ki slightly
- Verify minimum cooling speeds are adequate

#### Issue: Overshoot
**Symptoms**: Temperature goes well below setpoint
**Solutions**:
- Reduce ki
- Reduce integral_clamp_factor
- Increase cool_deadband value

#### Issue: Short Cycling
**Symptoms**: Compressor starts/stops frequently
**Solutions**:
- Increase min_cooling_off_time and min_cooling_run_time
- Check cool_deadband and cool_overrun settings
- Verify temperature sensor placement

#### Issue: High-Speed Never Engages
**Check**:
- `enable_high_speed` switch is ON
- Cooling demand high enough (>75% blower for 2+ minutes)
- No protection timers active

#### Issue: Fan Runs When System Off
**Check**:
- Recent compressor operation (protection mode)
- Pending shutdown requests due to short cycle protection
- Check system monitor logs for safety interventions

### Diagnostic Entity Reference
Monitor these entities during testing:
- `compressor_protection_remaining`: Shows protection timer
- `compressor_short_cycle_remaining`: Shows short cycle protection  
- `pid_target_speed` vs `pid_current_speed`: Shows ramp behavior
- `blower_speed_called`: Shows actual output to blower
- Binary sensors: Show system state and protection status

### Performance Targets
**Temperature Control**:
- Steady-state error: ±0.5°F
- Settling time: <15 minutes for 5°F change
- Overshoot: <2°F below setpoint

**Efficiency**:
- Compressor cycles: 3-6 per hour typical
- Minimum cycle time: 2 minutes (protected)
- Maximum blower speed during normal cooling: 60-80%

### Safety Validation Checklist
- [ ] Compressor protection delays function
- [ ] Short cycle protection prevents damage
- [ ] Minimum blower speeds enforced during cooling
- [ ] Emergency compressor-without-blower detection works
- [ ] System handles protection timer conflicts correctly
- [ ] Evaporator freeze protection active

## Final Validation Protocol

### System Acceptance Test
1. **Safety Systems**: All protection mechanisms function correctly
2. **Basic Operation**: All modes and fan speeds work as expected  
3. **PID Performance**: Temperature control meets targets
4. **Advanced Features**: High-speed compressor and protection systems work
5. **Integration**: Home Assistant control and monitoring complete
6. **Stability**: 24-hour operation without errors
7. **Edge Cases**: System handles unusual conditions gracefully

### Documentation Requirements
- Record final PID values and rationale
- Document any system-specific adjustments  
- Note environmental conditions during testing
- Save performance data for future reference
- Create operational procedures for users

Only mark system as fully validated when ALL tests pass and performance targets are met.