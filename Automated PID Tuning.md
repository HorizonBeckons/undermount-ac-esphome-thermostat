# Automated PID Data Collection System
**Untested

## Overview
This system provides automated data collection for PID tuning using Home Assistant automations, scripts, and data logging. It implements the testing methodology from the guide with minimal manual intervention.

## System Components

### 1. ESPHome Configuration Additions
### 2. Home Assistant Configuration  
### 3. Data Collection Scripts
### 4. Analysis Tools
### 5. Automated Test Procedures

---

## 1. ESPHome Configuration Additions

Add these components to your `undermount-ac.yaml` to enable automated testing:

```yaml
# Add to globals section
globals:
  # Testing control globals
  - id: test_mode_active
    type: bool
    initial_value: 'false'
  - id: test_start_time
    type: uint32_t
    initial_value: '0'
  - id: test_target_temp
    type: float
    initial_value: '0.0'
  - id: baseline_temp
    type: float
    initial_value: '0.0'

# Add to sensor section
sensor:
  # PID component sensors for detailed analysis
  - platform: template
    id: pid_error
    name: "PID Error"
    unit_of_measurement: "°C"
    accuracy_decimals: 3
    entity_category: diagnostic
    lambda: |-
      if (id(undermount_thermostat).mode == CLIMATE_MODE_COOL || 
          id(undermount_thermostat).mode == CLIMATE_MODE_FAN_ONLY) {
        float current = id(undermount_thermostat).current_temperature;
        float target = id(undermount_thermostat).target_temperature;
        return current - target;
      }
      return 0.0;

  - platform: template
    id: pid_proportional
    name: "PID Proportional Term"
    unit_of_measurement: "%"
    accuracy_decimals: 3
    entity_category: diagnostic
    lambda: |-
      float error = id(pid_error).state;
      return ${kp} * error * 100.0; // Convert to percentage

  - platform: template
    id: pid_integral_term
    name: "PID Integral Term"
    unit_of_measurement: "%"
    accuracy_decimals: 3
    entity_category: diagnostic
    lambda: |-
      return ${ki} * id(integral) * 100.0; // Convert to percentage

  - platform: template
    id: test_elapsed_time
    name: "Test Elapsed Time"
    unit_of_measurement: "min"
    accuracy_decimals: 1
    entity_category: diagnostic
    lambda: |-
      if (id(test_mode_active)) {
        return (millis() - id(test_start_time)) / 60000.0;
      }
      return 0.0;

  - platform: template
    id: system_performance_score
    name: "System Performance Score"
    accuracy_decimals: 2
    entity_category: diagnostic
    lambda: |-
      // Real-time performance scoring based on error and stability
      float error = fabs(id(pid_error).state);
      float target_speed = id(target_blower_speed);
      float current_speed = id(current_blower_speed);
      float speed_stability = 100.0 - fabs(target_speed - current_speed);
      
      // Score: 100 = perfect, 0 = poor
      float temp_score = fmax(0.0, 100.0 - (error * 50.0)); // 2°C error = 0 points
      float stability_score = fmax(0.0, speed_stability);
      
      return (temp_score + stability_score) / 2.0;

# Add to button section
button:
  - platform: template
    name: "Start PID Test"
    on_press:
      - lambda: |-
          ESP_LOGW("TEST", "=== STARTING AUTOMATED PID TEST ===");
          id(test_mode_active) = true;
          id(test_start_time) = millis();
          id(baseline_temp) = id(onboard_temperature).state;
          
          // Calculate test target (3°F below current)
          float target_celsius = id(baseline_temp) - 1.67; // 3°F = 1.67°C
          id(test_target_temp) = target_celsius;
          
          ESP_LOGW("TEST", "Baseline: %.2f°C, Target: %.2f°C", 
                   id(baseline_temp), target_celsius);
      - climate.control:
          id: undermount_thermostat
          mode: COOL
          target_temperature: !lambda "return id(test_target_temp);"
          fan_mode: AUTO

  - platform: template
    name: "Stop PID Test"
    on_press:
      - lambda: |-
          ESP_LOGW("TEST", "=== PID TEST STOPPED ===");
          id(test_mode_active) = false;
      - climate.control:
          id: undermount_thermostat
          mode: "OFF"

  - platform: template
    name: "Step Response Test"
    on_press:
      - lambda: |-
          ESP_LOGW("TEST", "=== STEP RESPONSE TEST ===");
          id(test_mode_active) = true;
          id(test_start_time) = millis();
          id(baseline_temp) = id(onboard_temperature).state;
          
          // Larger step change for step response (5°F)
          float target_celsius = id(baseline_temp) - 2.78; // 5°F = 2.78°C
          id(test_target_temp) = target_celsius;
          
          ESP_LOGW("TEST", "Step Response: %.2f°C -> %.2f°C", 
                   id(baseline_temp), target_celsius);
      - climate.control:
          id: undermount_thermostat
          mode: COOL
          target_temperature: !lambda "return id(test_target_temp);"
          fan_mode: AUTO

# Add to switch section  
switch:
  - platform: template
    name: "Enable Data Logging"
    id: enable_data_logging
    restore_mode: RESTORE_DEFAULT_OFF
    optimistic: true

# Enhanced interval for data collection
interval:
  - interval: 10s
    id: pid_data_logger
    then:
      - if:
          condition:
            and:
              - lambda: 'return id(test_mode_active);'
              - switch.is_on: enable_data_logging
          then:
            - lambda: |-
                // Log comprehensive PID data every 10 seconds during tests
                float elapsed_min = (millis() - id(test_start_time)) / 60000.0;
                
                ESP_LOGW("PID_DATA", "%.2f,%.3f,%.3f,%.3f,%.1f,%.1f,%.1f,%s,%s", 
                         elapsed_min,                           // Time
                         id(onboard_temperature).state,         // Current temp
                         id(undermount_thermostat).target_temperature, // Target temp  
                         id(pid_error).state,                   // Error
                         id(target_blower_speed),               // PID target speed
                         id(current_blower_speed),              // Current speed
                         id(blower_speed_called).state,         // Actual output
                         id(compressor_power).state ? "ON" : "OFF", // Compressor
                         id(compressor_speed_high).state ? "HIGH" : "NORMAL" // Speed mode
                );
```

---

## 2. Home Assistant Configuration

### Configuration.yaml additions:

```yaml
# Data logging for PID tuning
recorder:
  db_url: sqlite:////config/home-assistant_v2.db
  purge_keep_days: 90
  include:
    entities:
      - sensor.undermount_ac_temperature
      - sensor.undermount_ac_pid_error  
      - sensor.undermount_ac_pid_target_speed
      - sensor.undermount_ac_pid_current_speed
      - sensor.undermount_ac_output_1_blower_speed
      - sensor.undermount_ac_pid_proportional_term
      - sensor.undermount_ac_pid_integral_term
      - sensor.undermount_ac_test_elapsed_time
      - sensor.undermount_ac_system_performance_score
      - sensor.undermount_ac_compressor_protection_time_remaining
      - binary_sensor.undermount_ac_output_3_compressor_power
      - binary_sensor.undermount_ac_high_speed_status
      - climate.undermount_ac_air_conditioner

# Input helpers for test configuration
input_number:
  pid_test_duration:
    name: "PID Test Duration"
    min: 10
    max: 120
    step: 5
    unit_of_measurement: "min"
    initial: 30
    icon: mdi:timer

  pid_test_step_size:
    name: "PID Test Step Size"
    min: 1
    max: 10
    step: 0.5
    unit_of_measurement: "°F"
    initial: 3
    icon: mdi:thermometer

input_select:
  pid_test_type:
    name: "PID Test Type"
    options:
      - "Setpoint Response"
      - "Step Response"
      - "Load Disturbance"
      - "Stability Test"
    initial: "Setpoint Response"
    icon: mdi:test-tube

input_boolean:
  pid_auto_data_export:
    name: "Auto Export Data"
    initial: true
    icon: mdi:database-export

# Utility meter for test session tracking
utility_meter:
  daily_test_sessions:
    source: sensor.undermount_ac_test_elapsed_time
    cycle: daily
```

### Automations.yaml additions:

```yaml
# Automated PID Test Controller
- id: pid_test_controller
  alias: "PID Test Controller"
  description: "Manages automated PID testing sessions"
  trigger:
    - platform: state
      entity_id: button.undermount_ac_start_pid_test
  action:
    - service: notify.persistent_notification
      data:
        title: "PID Test Started"
        message: >
          Started {{ states('input_select.pid_test_type') }} test.
          Duration: {{ states('input_number.pid_test_duration') }} minutes.
          Step size: {{ states('input_number.pid_test_step_size') }}°F.
    
    - delay:
        minutes: "{{ states('input_number.pid_test_duration') | int }}"
    
    - service: button.press
      target:
        entity_id: button.undermount_ac_stop_pid_test
    
    - condition: state
      entity_id: input_boolean.pid_auto_data_export
      state: 'on'
    
    - service: script.export_pid_test_data

# Data export automation
- id: export_pid_data_on_test_complete
  alias: "Export PID Data on Test Complete"
  trigger:
    - platform: state
      entity_id: button.undermount_ac_stop_pid_test
  condition:
    - condition: state
      entity_id: input_boolean.pid_auto_data_export
      state: 'on'
  action:
    - delay:
        seconds: 5  # Let data settle
    - service: script.export_pid_test_data

# Performance monitoring during tests
- id: pid_performance_monitor
  alias: "PID Performance Monitor"
  trigger:
    - platform: numeric_state
      entity_id: sensor.undermount_ac_system_performance_score
      below: 70
      for:
        minutes: 5
  condition:
    - condition: state
      entity_id: switch.undermount_ac_enable_data_logging
      state: 'on'
  action:
    - service: notify.persistent_notification
      data:
        title: "PID Performance Alert"
        message: >
          Performance score below 70 for 5+ minutes.
          Current score: {{ states('sensor.undermount_ac_system_performance_score') }}
          Consider adjusting PID parameters.

# Safety monitoring during tests
- id: pid_test_safety_monitor
  alias: "PID Test Safety Monitor"
  trigger:
    - platform: numeric_state
      entity_id: sensor.undermount_ac_pid_error
      above: 5  # 5°C error indicates problem
      for:
        minutes: 10
  condition:
    - condition: state
      entity_id: switch.undermount_ac_enable_data_logging
      state: 'on'
  action:
    - service: button.press
      target:
        entity_id: button.undermount_ac_stop_pid_test
    - service: notify.persistent_notification
      data:
        title: "PID Test Safety Stop"
        message: >
          Test stopped due to large temperature error (>5°C).
          System may be unstable with current PID settings.
```

### Scripts.yaml additions:

```yaml
# Data export script
export_pid_test_data:
  alias: "Export PID Test Data"
  sequence:
    - service: recorder.purge
      data:
        keep_days: 1
        repack: false
        apply_filter: false
    
    - delay:
        seconds: 2
    
    - service: notify.persistent_notification
      data:
        title: "PID Data Export"
        message: >
          PID test data exported to: /config/pid_test_data_{{ now().strftime('%Y%m%d_%H%M%S') }}.csv
          Use the data analysis scripts to process results.

# Automated tuning sequence
run_pid_tuning_sequence:
  alias: "Run PID Tuning Sequence" 
  sequence:
    # Phase 1: Proportional gain sweep
    - repeat:
        count: 4
        sequence:
          - service: esphome.undermount_ac_update_pid_kp
            data:
              kp: "{{ 0.2 + (repeat.index - 1) * 0.2 }}"  # 0.2, 0.4, 0.6, 0.8
          
          - delay:
              minutes: 2  # Let system stabilize
          
          - service: button.press
            target:
              entity_id: button.undermount_ac_start_pid_test
          
          - delay:
              minutes: "{{ states('input_number.pid_test_duration') | int }}"
          
          - service: script.export_pid_test_data
          
          - delay:
              minutes: 5  # Cool down between tests

# Quick performance check
quick_pid_check:
  alias: "Quick PID Performance Check"
  sequence:
    - service: input_number.set_value
      target:
        entity_id: input_number.pid_test_duration
      data:
        value: 15  # Short 15-minute test
    
    - service: input_select.select_option
      target:
        entity_id: input_select.pid_test_type
      data:
        option: "Setpoint Response"
    
    - service: button.press
      target:
        entity_id: button.undermount_ac_start_pid_test
```

---

## 3. Data Collection Scripts

### Python Data Analysis Script (`/config/python_scripts/pid_analyzer.py`):

```python
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
from datetime import datetime, timedelta
import sqlite3
import os

class PIDAnalyzer:
    def __init__(self, ha_db_path="/config/home-assistant_v2.db"):
        self.db_path = ha_db_path
        
    def extract_test_data(self, start_time, end_time):
        """Extract PID test data from Home Assistant database"""
        conn = sqlite3.connect(self.db_path)
        
        query = """
        SELECT 
            datetime(s.last_updated, 'localtime') as timestamp,
            s.entity_id,
            s.state
        FROM states s
        WHERE s.entity_id IN (
            'sensor.undermount_ac_temperature',
            'sensor.undermount_ac_pid_error',
            'sensor.undermount_ac_pid_target_speed',
            'sensor.undermount_ac_pid_current_speed',
            'sensor.undermount_ac_output_1_blower_speed',
            'sensor.undermount_ac_pid_proportional_term',
            'sensor.undermount_ac_pid_integral_term',
            'sensor.undermount_ac_system_performance_score',
            'binary_sensor.undermount_ac_output_3_compressor_power'
        )
        AND s.last_updated BETWEEN ? AND ?
        AND s.state NOT IN ('unavailable', 'unknown')
        ORDER BY s.last_updated
        """
        
        df = pd.read_sql_query(query, conn, params=[start_time, end_time])
        conn.close()
        
        # Pivot data for analysis
        df_pivot = df.pivot(index='timestamp', columns='entity_id', values='state')
        df_pivot.index = pd.to_datetime(df_pivot.index)
        
        return df_pivot
    
    def analyze_step_response(self, df):
        """Analyze step response characteristics"""
        if df.empty:
            return None
        
        # Convert columns to numeric where possible
        numeric_cols = [
            'sensor.undermount_ac_temperature',
            'sensor.undermount_ac_pid_error', 
            'sensor.undermount_ac_pid_target_speed',
            'sensor.undermount_ac_pid_current_speed',
            'sensor.undermount_ac_output_1_blower_speed',
            'sensor.undermount_ac_system_performance_score'
        ]
        
        for col in numeric_cols:
            if col in df.columns:
                df[col] = pd.to_numeric(df[col], errors='coerce')
        
        error = df['sensor.undermount_ac_pid_error']
        temp = df['sensor.undermount_ac_temperature']
        
        # Calculate metrics
        analysis = {
            'max_error': abs(error).max(),
            'final_error': abs(error.iloc[-10:]).mean(),  # Last 10 readings
            'settling_time': self._calculate_settling_time(error),
            'rise_time': self._calculate_rise_time(temp),
            'overshoot': self._calculate_overshoot(temp),
            'mean_performance_score': df['sensor.undermount_ac_system_performance_score'].mean(),
            'stability_metric': error.std()
        }
        
        return analysis
    
    def _calculate_settling_time(self, error, tolerance=0.5):
        """Calculate time to settle within tolerance"""
        abs_error = abs(error)
        settled_mask = abs_error <= tolerance
        
        # Find first point where it settles and stays settled
        for i in range(len(settled_mask)-10):
            if settled_mask.iloc[i:i+10].all():  # Settled for 10 consecutive readings
                return i * 10 / 60  # Convert to minutes (10s intervals)
        
        return None
    
    def _calculate_rise_time(self, temp):
        """Calculate rise time (10% to 90% of final value)"""
        if len(temp) < 10:
            return None
            
        initial = temp.iloc[0]
        final = temp.iloc[-10:].mean()
        change = final - initial
        
        ten_percent = initial + 0.1 * change
        ninety_percent = initial + 0.9 * change
        
        try:
            t10_idx = temp[temp <= ten_percent].index[-1]
            t90_idx = temp[temp <= ninety_percent].index[0]
            
            rise_time = (t90_idx - t10_idx).total_seconds() / 60
            return rise_time
        except:
            return None
    
    def _calculate_overshoot(self, temp):
        """Calculate maximum overshoot below target"""
        if len(temp) < 10:
            return 0
            
        final_temp = temp.iloc[-10:].mean()
        min_temp = temp.min()
        
        return max(0, final_temp - min_temp)
    
    def generate_report(self, test_data, analysis, output_file):
        """Generate comprehensive test report"""
        report = f"""
PID Test Analysis Report
Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
{'='*50}

Test Duration: {len(test_data)} data points ({len(test_data)*10/60:.1f} minutes)

PERFORMANCE METRICS:
- Maximum Error: {analysis['max_error']:.2f}°C
- Final Steady-State Error: {analysis['final_error']:.2f}°C  
- Settling Time: {analysis['settling_time']:.1f} minutes
- Rise Time: {analysis['rise_time']:.1f} minutes
- Overshoot: {analysis['overshoot']:.2f}°C
- Mean Performance Score: {analysis['mean_performance_score']:.1f}/100
- Stability Metric (StdDev): {analysis['stability_metric']:.3f}

PERFORMANCE ASSESSMENT:
"""
        
        # Performance scoring
        score = 0
        if analysis['final_error'] < 0.5:
            score += 25
            report += "✓ Excellent steady-state accuracy\n"
        elif analysis['final_error'] < 1.0:
            score += 15
            report += "○ Good steady-state accuracy\n"
        else:
            report += "✗ Poor steady-state accuracy\n"
            
        if analysis['settling_time'] and analysis['settling_time'] < 15:
            score += 25
            report += "✓ Fast settling time\n"
        elif analysis['settling_time'] and analysis['settling_time'] < 30:
            score += 15
            report += "○ Acceptable settling time\n"
        else:
            report += "✗ Slow settling time\n"
            
        if analysis['overshoot'] < 1.0:
            score += 25
            report += "✓ Minimal overshoot\n"
        elif analysis['overshoot'] < 2.0:
            score += 15
            report += "○ Moderate overshoot\n"
        else:
            report += "✗ Excessive overshoot\n"
            
        if analysis['stability_metric'] < 0.5:
            score += 25
            report += "✓ Excellent stability\n"
        elif analysis['stability_metric'] < 1.0:
            score += 15
            report += "○ Good stability\n"
        else:
            report += "✗ Poor stability (oscillations)\n"
            
        report += f"\nOVERALL SCORE: {score}/100\n"
        
        if score >= 80:
            report += "RECOMMENDATION: PID tuning is excellent\n"
        elif score >= 60:
            report += "RECOMMENDATION: PID tuning is acceptable, minor adjustments may help\n"
        elif score >= 40:
            report += "RECOMMENDATION: PID tuning needs improvement\n"
        else:
            report += "RECOMMENDATION: PID tuning requires significant adjustment\n"
        
        with open(output_file, 'w') as f:
            f.write(report)
        
        return report

# Usage example
if __name__ == "__main__":
    analyzer = PIDAnalyzer()
    
    # Analyze last hour of data
    end_time = datetime.now()
    start_time = end_time - timedelta(hours=1)
    
    df = analyzer.extract_test_data(start_time, end_time)
    analysis = analyzer.analyze_step_response(df)
    
    if analysis:
        report = analyzer.generate_report(df, analysis, f"/config/pid_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt")
        print(report)
```

---

## 4. Dashboard Configuration

### Lovelace dashboard (`pid_tuning_dashboard.yaml`):

```yaml
title: PID Tuning Dashboard
path: pid-tuning
icon: mdi:tune
cards:
  - type: vertical-stack
    cards:
      - type: entities
        title: Test Control
        entities:
          - input_select.pid_test_type
          - input_number.pid_test_duration
          - input_number.pid_test_step_size
          - input_boolean.pid_auto_data_export
          - switch.undermount_ac_enable_data_logging
          
      - type: horizontal-stack
        cards:
          - type: button
            entity: button.undermount_ac_start_pid_test
            name: Start Test
            icon: mdi:play
            tap_action:
              action: call-service
              service: button.press
              target:
                entity_id: button.undermount_ac_start_pid_test
                
          - type: button  
            entity: button.undermount_ac_stop_pid_test
            name: Stop Test
            icon: mdi:stop
            tap_action:
              action: call-service
              service: button.press
              target:
                entity_id: button.undermount_ac_stop_pid_test

  - type: history-graph
    title: Temperature Control
    hours_to_show: 2
    refresh_interval: 30
    entities:
      - entity: sensor.undermount_ac_temperature
        name: Current Temperature
      - entity: climate.undermount_ac_air_conditioner
        attribute: temperature
        name: Target Temperature
      - entity: sensor.undermount_ac_pid_error
        name: Error

  - type: history-graph
    title: Blower Speed Control
    hours_to_show: 2
    refresh_interval: 30
    entities:
      - entity: sensor.undermount_ac_pid_target_speed
        name: PID Target
      - entity: sensor.undermount_ac_pid_current_speed  
        name: Current Speed
      - entity: sensor.undermount_ac_output_1_blower_speed
        name: Actual Output

  - type: history-graph
    title: PID Components
    hours_to_show: 2
    refresh_interval: 30
    entities:
      - entity: sensor.undermount_ac_pid_proportional_term
        name: Proportional
      - entity: sensor.undermount_ac_pid_integral_term
        name: Integral

  - type: entities
    title: System Status
    entities:
      - sensor.undermount_ac_test_elapsed_time
      - sensor.undermount_ac_system_performance_score
      - binary_sensor.undermount_ac_output_3_compressor_power
      - binary_sensor.undermount_ac_high_speed_status
      - sensor.undermount_ac_compressor_protection_time_remaining

  - type: gauge
    entity: sensor.undermount_ac_system_performance_score
    title: Performance Score
    min: 0
    max: 100
    severity:
      green: 80
      yellow: 60
      red: 0
```

---

## 5. Usage Instructions

### Quick Start Automated Testing:

1. **Setup**: 
   - Add ESPHome configurations and recompile
   - Add Home Assistant configurations and restart
   - Enable all diagnostic sensors in HA

2. **Run Basic Test**:
   - Go to PID Tuning Dashboard
   - Set test duration (start with 30 minutes)
   - Enable data logging
   - Press "Start Test" button
   - Monitor progress on dashboard

3. **Data Analysis**:
   - After test completes, data is automatically exported
   - Run Python analysis script for detailed metrics
   - Review performance score and recommendations

### Automated Tuning Sequence:

1. **Proportional Gain Sweep**:
   ```yaml
   # Run this script in HA
   service: script.run_pid_tuning_sequence
   ```

2. **Analysis**:
   - System automatically tests kp values: 0.2, 0.4, 0.6, 0.8
   - Each test runs for configured duration
   - Data exported after each test
   - Compare performance scores to find optimal kp

3. **Integral Tuning**:
   - Use best kp from sweep
   - Manually adjust ki values and run individual tests
   - Monitor for steady-state error elimination

### Key Monitoring Points:

- **Performance Score**: Real-time system assessment
- **Temperature Error**: Shows control accuracy  
- **Speed Stability**: Indicates smooth operation
- **Protection Status**: Ensures safety systems active

### Data Export Format:

The system exports CSV data with columns:
- Timestamp
- Current Temperature  
- Target Temperature
- PID Error
- Target Blower Speed
- Current Blower Speed
- Actual Output
- Compressor Status
- High-Speed Status

This automated system eliminates manual data collection and provides scientific, repeatable PID tuning with comprehensive analysis and safety monitoring.