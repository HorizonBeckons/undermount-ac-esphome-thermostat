![image](https://github.com/user-attachments/assets/f090b563-4deb-4ac4-91a9-82fff0fcb047)

# Undermount AC ESPHome Thermostat

This project implements the [Undermount AC ESPHome Thermostat](https://undermountac.com/pages/hass) for the V3 [Undermount AC](https://undermountac.com/) system and replaces the standard thermostat allowing the user to control the thermostat directly from [Home Assistant](https://www.home-assistant.io/).  It also adds additional capabilities not found in the standard thermostat.

The design goals for this project are:

1. Maximize the comfort of the occupants by closely maintaining the set temperature.
2. Minimize the noise generated from the blower.
3. Maximize the efficiency of the system.

> **Disclaimer:** The Software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement. In no event shall the author be liable for any claim, damages, or other liability arising from the use of the Software.  This project is **not** associated or endorsed by Undermount AC.

## Main Features

* Control your Undermount AC via Home Assistant
* Low/Med/High manual fan control 
* Enable/Disable High-speed Compressor
* Dynamic blower speed to maintain your set temperature - \*New\*
* Dynamic high-speed compressor control - \*New\*
* Soft blower speed ramp up/down - \*New\*

## Prerequisites

You'll need an **ESPHome Thermostat by Undermount AC**.

You can purchase that from the Undermount AC website [here](https://undermountac.com/products/esphome-thermostat-controller-12-30v-dc)[.](https://undermountac.com/products/esphome-thermostat-controller-12-30v-dc)

> ℹ️ **Note:** These instructions assume you know about Home Assistant and the ESPHome add-on.

# Installation

Installation for the thermostat is straight forward.  At a high-level it is as follows:

1. Program the thermostat
2. Physically install it in the van
3. Connect and configure in Home Assistant
4. Understand and test operation

Let's get started! 🚀

## Program Thermostat

The ESPHome firmware and configuration need to be loaded on the thermostat.

Remove the top cover and connect the **UART interface** on the Undermount AC Thermostat to your computer using a USB cable.

Use the "Install Firmware" link below to install the **pre-built firmware** directly to the ESPHome HVAC Controller via USB from Chromium-based browser.

## Install Undermount AC Thermostat

[![ESPHome Install](https://img.shields.io/badge/Install%20ESPHome-blue?style=for-the-badge&logo=esphome)](https://esphome.github.io/esphome-flasher/?configuration=https://raw.githubusercontent.com/anthonysecco/undermount-ac-esphome-thermostat/main/undermount-ac.yaml)

<esp-web-install-button manifest="firmware/anthonysecco.undermount-ac-esphome-thermostat.manifest.json.manifest.json"></esp-web-install-button>

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

Once installed, a red LED should start blinking on the device, and a wireless network named **"Undermount AC Failover AP"** will broadcast from the thermostat.

The thermostat is now ready for physical installation.  Disconnect the USB cable, reattached the cover, and head to the RV.

## Physical Installation

Locating the best location for the thermostat is important for accurate and consistent readings.  The temperature probe at the end of the 3 meter cable from the thermostat should be placed 1.25-1.5M or 4-5FT from the floor, away from a window or door and not on a surface attached to the exterior of the van.

Once you identify the location, mount the thermostat controller and the temperature probe.  Now you're ready to wire it up.

### Hardware & Wiring

> ⚠️ **Warning:** Power should be disconnected from the Undermount system during thermostat installation.

Wire the thermostat according to the table below for the V3 system:

| Terminal | Jumper Setting | Wire           | Description                 |
| -------- | -------------- | -------------- | --------------------------- |
| Neg      | N/A            | **Black Wire** | Ground connection           |
| Pos      | N/A            | **Red Wire**   | Power supply (10-31 VDC)    |
| Output 1 | High (H)       | **White Wire** | PWM Blower                  |
| Output 2 | High (H)       | **Brown Wire** | Voltage to Blower Fan Relay |
| Output 3 | High (H)       | **Blue Wire**  | Engage Cool Relay           |
| Output 4 | High (H)       | **Green Wire** | High Speed Compressor Relay |
| Output 5 | N/A            | Unused         | Available for other uses    |
| Output 6 | N/A            | Unused         | Available for other uses    |

Output 5 & 6 are unused and can be configured for other purposes.  They are ground-switched and can drive up to 1A of current. Further hardware specifications can be found on the Undermount website here.

## Connect & Configure

### Connect to Wi-Fi

With the wiring complete, apply power to the Undermount AC system.

* The onboard RGD LED should blink red and a wireless network "**Undermount AC Failover AP**" should become available.
* Using your computer, connect to "**Undermount AC Failover AP**" using the password: 12345678.
* A **captive portal** will load. Select the Wi-Fi network where your **Home Assistant** server is running and provide your credentials.
* Once applied, the thermostat will reboot and connect to the target Wi-Fi network.  The LED will stop blinking red if connected.

### Configure Home Assistant

If you don’t have ESPHome installed in Home Assistant, [follow this guide](https://esphome.io/guides/getting_started_command_line.html) to install it.

* Once the thermostat is connected to Wi-Fi, you should see it as a **newly discovered ESPHome device** under **Settings → Devices & Services** in Home Assistant.
* Click **Configure**, and the device will be added to Home Assistant.

You should now see the **Undermount AC** Climate Component in Home Assistant.

### Adopt in ESPHome (Optional)

In your ESPHome dashboard, you should see the device with the option to **“Adopt”** (this may require restarting Home Assistant). Click **Adopt**, and you’ll be able to rename the device and install the updated configuration. From this point on, you have full control over the YAML configuration of the thermostat.

## Entities

You should now see the thermostat in Home Assistant.  Go to **Settings >> Devices >> ESPHome >> Undermount AC** and you should see the following.  Diagnostic sensors will be disabled by default.

### Controls

| Entity                       | Type                                                             |
| ---------------------------- |----------------------------------------------------------------- |
| Air Conditioner              | Main climate control component    |
| Enable High-speed Compressor | Allow high-speed compressor to be called under high loads |
| Blower                       | Disabled by default.  Allows manual control of blower |
| Minimum Cooling Blower Speed | Sets minimum blower speed while cooling. Range (40-60%)

### Sensors

| Entity                       | Type                                                             |
| ---------------------------- |----------------------------------------------------------------- |
| Temperature                  | Temperature from thermostat probe   |
| Humidity                     | Humdidity from thermostat probe  |
| High-Speed Status            | Indicates if compressor is in high-speed mode |

### Diagnostics

| Entity                       | Type                                                             |
| ---------------------------- |----------------------------------------------------------------- |
| Output 1 (Blower Speed)      | Indicates the speed (0-100%) being called   |
| Output 2 (Blower Power)      | Indicate if blower 'on' is being called  |
| Output 3 (Compressor Power)  | Indicates if compressor is being called |
| Ramp Blower                  | Indicates if blower is changing speed |

# Operations

The following is not an exhaustive list of how the climate component works in ESPHome.  Instread this provides details specific to this system.

## Fan Modes
The climate component refers to the blower as 'fan'.  References to 'blower' and 'fan' are one in the same.  The fan has four operating modes.  

| Mode | Description
| ----- | ---------- |
| Low | Minimum blower speed |
| Medium | Medium blower speed |
| High | High blower speed |
| Auto | Dynamic blower speed.  Varies based on system demand |

### Auto Mode
The Auto Mode engages dynamic blower speed.  This blower adjusts speed automatically to maintain the desired target temperature.  This is the recommended mode and should be a "set it and forget it" configuration.

> Dynamic blower speed is driven through PID control loop logic.  Future versions will exposed tuning controls in Home Assitant.

### Slow Ramping
The blower will change speeds slowly to minimize audible changes to occupants.  This includes transistion from Off, Idle, and Cooling modes.

## Thermostat Modes
The thermostat has three modes.

| Mode | Description
| ----- | ---------- |
| OFF | All systems are turned off |
| COOLING | System active cooling.  Compressor and blower are active. |
| FAN ONLY | Only blower is active. |
| IDLE | Actual temperature is below set point.  Transistory state. |

### Status Light

The physical unit will show the following status lights when in operation.

| Light Behavior    | Description                         | 
| ----------------- | ----------------------------------- |
| Flashing Red      | Not connected to Home Assistant     |
| Faint Blue        | IDLE |
| Slow Pulsing Blue | COOLING                             |
| Fast Pulsing Blue | COOLING - High-Speed Compressor     |
| Slow Pulsing Cyan | FAN ONLY                            |
| Steady Orange     | OFF             |

### OFF
This is self-explainatory.  System will remain off until switched back to another mode.

When transitioning into OFF, the blower will ramp down to minimum speed and remain on for 30 seconds to push residual cool air.  The blower will remain off for the duration of the OFF state.

> Running the blower on idle for >30s will likely start to push condesation on the evaporator coils back into the air.

### Cooling
Cooling will start when the actual temperature is 0.5C above the set temperature.  When cooling starts, the compressor will immediately be called.  

> The blower has a delayed start of 15 seconds to allow the evaporator to cool slightly before pushing air.

#### High-Speed Cooling
If enabled, the thermostat will engage the high-speed compressor mode under high cooling demands.

When the Enable High-Speed Cooling switch is set to **ON** in Home Assistant, the compressor will switch into High-Speed mode above 75% for two minutes.  It will remain in High-speed until the blower drops below 65% for two minutes.  In effect, high-speed compressor is only engaged when the fan is in **HIGH** mode or **AUTO** mode and the system is demanding >75% blower speed.

If the High-Speed Cooling switch is set to **OFF**, low speed compressor mode will be used regardless of blower speed.

> ℹ️ **Note:** the Undermount AC compressor has a total of three speeds, however, in daily operation only two can be activated, Low or (Medium/High).  Most users find Medium (the default) to be sufficent.  Refer to Undermount AC documentation for details.

### IDLE
The is a transistory state and is not selectable.  When in COOLING or FAN ONLY mode and the actual temperature is 1C below the set temperature, the thermostat will transistion to IDLE state.

When transitioning into IDLE, the blower will slowly ramp down to minimum speed and run for 30 seconds to push out residual cool air.  The blower will remain off for the duration of the IDLE state.

> Running the blower on idle for >30s will likely start to push condesation on the evaporator coils back into the air.

## Recommended Settings
Most users will get the best results setting the fan to **AUTO** and High-Speed Compressor to **ON**.

## System Protections
The thermostat has built-in protections to keep equipment opearting correctly

### Compressor Runtime
The compressor will remain **OFF** or **ON** for at least 2 minutes.  In addition, the compressor will not change speeds for at least 2 minutes.  This protects from short cycling and frequent pressure changes in the coolant.

### Minimum Blower Speed
The *minmum* speed will increase in **Cooling Mode** to prevent the evaporator from freezing.  This should be no less than 40%, but may be increased if necessary. 
 This can be increased in the YAML configuration if evaporator freezing becomes a problem.

# Conclusion

At this point, you've completed the installation.  You may create your dashboard and setup automations to your heart's content.  

## Contact

Please raise an issue for enhancements, bugs, or any other questions regarding this project.  Thanks!

⭐ Star this project if you found it useful.

## Contribution

> Special thanks to Mike from SmartyVan for the initial code on this thermostat [here](https://github.com/mikegoubeaux/UndermountAC).

If you would like to contribute:

1. Fork the repository.
2. Create a new branch: `git checkout -b feature-name`
3. Make your changes and commit: `git commit -m 'Add feature'`
4. Push to the branch: `git push origin feature-name`
5. Submit a pull request.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details.
