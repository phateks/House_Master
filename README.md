# House Master - ESPHome Modular Configuration

## Overview
This is a modular ESPHome configuration for home automation, featuring:
- Multiple modbus thermostats (easily expandable)
- Multiple light switches and dimmers
- Binary sensors
- PWM outputs

## Structure

### Main Files
- **house_master.yaml** - Main configuration file (WiFi, MQTT, API, packages)
- **hardware.yaml** - Hardware definition package (I2C, MCP23017, PWM, 70 inputs, 16 lights) - **FIXED, do not modify**
- **modbus_thermostats.yaml** - Reusable thermostat package
- **io_extension.yaml** - Reusable I/O extension package for connecting inputs to lights
- **example_16_lights.yaml** - Example configurations showing how to connect inputs to lights (ready to copy/paste)

## Hardware Layer (FIXED)

The [hardware.yaml](hardware.yaml) file defines all physical components of the Kincony KC868-A4S board:
- **UART** (GPIO32/GPIO33) for Modbus communication
- **Modbus hub** (for thermostats)
- **I2C bus** (SDA=GPIO4, SCL=GPIO16)
- **5x MCP23017** I/O expanders (70 inputs total: `ext_in1` to `ext_in70`)
- **2x PCF8574** I/O expanders (additional inputs/outputs `a4s_input1-12`, `light1-4`)
- **1x PCA9685** PWM driver (16 channels: `pwm_0` to `pwm_15`)
- **16 Light entities** (`pwm_0` to `pwm_15`) connected to PWM outputs for 0-10V dimming
- **4 Onboard relays** (`relay1` to `relay4`) on dedicated GPIO pins

**This layer is included via packages and should NOT be modified.**

```yaml
# In house_master.yaml
packages:
  hardware: !include hardware.yaml
```

## Configuration Layer (FLEXIBLE)

## Adding New I/O Extensions (Dimming Lights)

The I/O extension package **connects** 2 existing input sensors to 1 existing light entity. It does NOT create hardware - it only creates the automation logic.

**Prerequisites:**
- Your inputs must already be defined as binary sensors (e.g., `ext_in1`, `ext_in2`, ... `ext_in70`)
- Your lights/dimmers must already be defined as light entities (e.g., `pwm_0`, `pwm_1`, `a4s_color_led_7`, etc.)

**Functionality:**
- **Short press** on either input = Toggle light on/off
- **Long press** on first input = Dim UP continuously
- **Long press** on second input = Dim DOWN continuously

### Example Configuration:

```yaml
packages:
  living_light: !include
    file: io_extension.yaml
    vars:
      input_id_up: "ext_in23"      # Choose ANY input for dim up
      input_id_down: "ext_in24"    # Choose ANY input for dim down
      light_id: "pwm_0"            # Choose ANY existing light
      dim_step: "5%"               # Optional: dimming step size
      dim_delay: "0.1s"            # Optional: delay between steps
```

### Parameters:
- **input_id_up**: ID of existing binary sensor for dim up (e.g., `ext_in23`)
- **input_id_down**: ID of existing binary sensor for dim down (e.g., `ext_in24`)
- **light_id**: ID of existing light entity (e.g., `pwm_0`, `a4s_color_led_7`)
- **dim_step**: Dimming step size (default: 5%)
- **dim_delay**: Delay between dimming steps (default: 0.1s)

### Example: Multiple Light Controls

```yaml
packages:
  # Living room: ext_in23 & ext_in24 control pwm_0
  living_light: !include
    file: io_extension.yaml
    vars:
      input_id_up: "ext_in23"
      input_id_down: "ext_in24"
      light_id: "pwm_0"
  
  # Bedroom: ext_in1 & ext_in2 control pwm_1
  bedroom_light: !include
    file: io_extension.yaml
    vars:
      input_id_up: "ext_in1"
      input_id_down: "ext_in2"
      light_id: "pwm_1"
  
  # Kitchen: ext_in45 & ext_in46 control a4s_color_led_7
  kitchen_light: !include
    file: io_extension.yaml
    vars:
      input_id_up: "ext_in45"
      input_id_down: "ext_in46"
      light_id: "a4s_color_led_7"
      dim_step: "3%"               # Finer dimming control
```

**Key Advantage**: Total flexibility! You can choose ANY 2 inputs from your 70 available inputs and connect them to ANY light. No need to follow a specific pin mapping.

## Adding New Thermostats

The thermostat configuration is fully modular. To add a new thermostat, simply add a new package entry in `house_master.yaml`:

```yaml
packages:
  thermostat_05: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_05"
      thermo_name: "Bathroom Thermostat"
      thermo_address: "5"
```

### Parameters:
- **thermo_id**: Unique identifier (e.g., `thermo_05`, `thermo_06`)
- **thermo_name**: Friendly name that appears in Home Assistant
- **thermo_address**: Modbus address (must match physical device address)

## Using Configuration from GitHub

To use these modular configurations directly from GitHub in another ESPHome project:

### Hardware (complete hardware layer):
```yaml
packages:
  hardware: !include
    github://YOUR_USERNAME/House_Master/hardware.yaml@main
```

### Thermostats:
```yaml
packages:
  thermostat_bedroom: !include
    github://YOUR_USERNAME/House_Master/modbus_thermostats.yaml@main
    vars:
      thermo_id: "thermo_bedroom"
      thermo_name: "Bedroom Thermostat"
      thermo_address: "2"
```

### I/O Extensions (Light Controls):
```yaml
packages:
  bedroom_light: !include
    github://YOUR_USERNAME/House_Master/io_extension.yaml@main
    vars:
      input_id_up: "ext_in23"
      input_id_down: "ext_in24"
      light_id: "pwm_0"
      dim_step: "5%"
      dim_delay: "0.1s"
```

Replace `YOUR_USERNAME` with your GitHub username and `main` with your branch name.

## Thermostat Features

Each thermostat includes:
- **Room Temperature Sensor** - Current temperature reading
- **Target Temperature Sensor** - Current setpoint (read-only)
- **Set Temperature Control** - Number input to set desired temperature (15-35°C)
- **Power Switch** - On/Off control
- **Climate Entity** - Full thermostat control in Home Assistant

## I/O Extension Features

Each I/O extension package creates automation logic that:
- **Connects** 2 existing inputs to 1 existing light (no hardware creation)
- **Short press** (50-350ms) on either input → Toggle light on/off
- **Long press** on first input → Continuous dim up (while held)
- **Long press** on second input → Continuous dim down (while held)
- **Configurable Dimming** - Adjustable step size and speed
- **Maximum Flexibility** - Choose ANY inputs (from ext_in1 to ext_in70) and ANY light

## Example: Adding 7 More Thermostats

```yaml
packages:
  # Existing thermostats
  thermostat_02: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_02"
      thermo_name: "Bedroom 1"
      thermo_address: "2"
  
  thermostat_03: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_03"
      thermo_name: "Living Room"
      thermo_address: "3"
  
  thermostat_04: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_04"
      thermo_name: "Kitchen"
      thermo_address: "4"
  
  # New thermostats
  thermostat_05: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_05"
      thermo_name: "Bedroom 2"
      thermo_address: "5"
  
  thermostat_06: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_06"
      thermo_name: "Bathroom"
      thermo_address: "6"
  
  thermostat_07: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_07"
      thermo_name: "Office"
      thermo_address: "7"
  
  thermostat_08: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_08"
      thermo_name: "Guest Room"
      thermo_address: "8"
  
  thermostat_09: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_09"
      thermo_name: "Hallway"
      thermo_address: "9"
  
  thermostat_10: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_10"
      thermo_name: "Garage"
      thermo_address: "10"
  
  thermostat_11: !include
    file: modbus_thermostats.yaml
    vars:
      thermo_id: "thermo_11"
      thermo_name: "Attic"
      thermo_address: "11"
```

## Hardware Configuration

### ESP32 Board
- Board: esp32dev
- UART: TX=GPIO32, RX=GPIO33
- Baud Rate: 9600

### Modbus
- ID: modbus1
- Connected thermostats use addresses 1-11 (expandable)

### I2C Components
- SDA: GPIO4
- SCL: GPIO16

## Notes
- Each thermostat package is completely self-contained
- No conflicts between thermostats - each has unique IDs
- Easy to add or remove thermostats without touching existing code
- All settings (min/max temp, step size, etc.) can be customized per thermostat if needed

## License
MIT
Kincony A4S house master controller 
