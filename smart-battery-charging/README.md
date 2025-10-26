# Smart Battery Charging System for Home Assistant

An intelligent automation system that optimizes home battery charging using the cheapest electricity periods while ensuring efficient energy management during autumn and winter months.

## Overview

This system automatically:
- Calculates the optimal charging time window based on battery deficit and available cheap electricity periods
- Dynamically adjusts charging power to match available time slots
- Uses seasonal logic (active October-March)
- Stops charging when battery reaches 100% or cheap period ends
- Provides detailed Telegram notifications

## Files

1. **`smart_battery_charging_config.yaml`** - Essential configuration parameters
2. **`smart_battery_charging_automation.yaml`** - Main automation logic with embedded calculations

## Prerequisites

### Required Integration

This system requires the **Czech Energy Spot Prices** integration to provide electricity price sensors and binary sensors for cheapest time periods.

**Integration:** [Home Assistant Czech Energy Spot Prices](https://github.com/rnovacek/homeassistant_cz_energy_spot_prices)

**Description:** Home Assistant integration that provides current Czech electricity spot prices based on OTE (Czech market operator). It includes sensors for monitoring current, cheapest, and most expensive electricity prices, and is compatible with Home Assistant automations for energy optimization.

**Installation:**
1. Install via HACS: Search for "Czech Energy Spot Prices" and install it
2. Or manually: Download and copy `custom_components/cz_energy_spot_prices` to your HA config
3. Add the integration via Settings → Devices & Services → Add integration
4. Configure currency (CZK) and energy unit (kWh)

This integration provides all the electricity price sensors and binary sensors used by the smart charging automation.

## Installation

### Using Packages (Recommended)

1. Create a `packages/` folder in your Home Assistant configuration directory
2. Create `packages/smart_battery_charging.yaml` with the combined content:

```yaml
# Smart Battery Charging Package
input_number:
  smart_charging_max_power:
    name: "Smart Charging Max Power"
    min: 1000
    max: 45000
    step: 100
    unit_of_measurement: "W"
    icon: mdi:lightning-bolt
    initial: 4000

input_boolean:
  smart_charging_enabled:
    name: "Smart Charging Enabled"
    icon: mdi:battery-charging-wireless
    initial: true

automation:
  # Copy the entire automation content from smart_battery_charging_automation.yaml here
```

3. Enable packages in your `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

4. Restart Home Assistant
5. Configure the input helpers in the UI if needed

## Configuration

### Input Helpers

- **Smart Charging Max Power** (`input_number.smart_charging_max_power`)
  - Range: 1000-4500W (default: 4000W)
  - Maximum charging power for your battery system

- **Smart Charging Enabled** (`input_boolean.smart_charging_enabled`)
  - Enable/disable the entire smart charging system

## How It Works

### Smart Time Window Selection

The system calculates required charging time and selects the optimal electricity price window:

- **≤1 hour needed** → Uses cheapest single hour
- **≤2 hours needed** → Uses cheapest 2-hour block
- **≤3 hours needed** → Uses cheapest 3-hour block
- **≤4 hours needed** → Uses cheapest 4-hour block
- **≤6 hours needed** → Uses cheapest 6-hour block
- **>6 hours needed** → Uses cheapest 8-hour block

### Dynamic Power Adjustment

Charging power is automatically adjusted based on available time:
- **1-hour window**: Full power up to deficit amount
- **2-hour window**: Deficit ÷ 2 (max configured power)
- **3-hour window**: Deficit ÷ 3 (max configured power)
- **And so on...**

### Seasonal Logic

- **Active**: October through March (autumn/winter)
- **Inactive**: April through September (spring/summer)

## Required Sensors

Your Home Assistant must have these sensors configured (provided by the Czech Energy Spot Prices integration):

### Battery Sensors
- `sensor.solax_bms_battery_capacity` - Total battery capacity in Wh
- `sensor.solax_remaining_battery_capacity` - Current battery level in Wh

### Electricity Price Sensors (from Czech Energy Spot Prices integration)
- `binary_sensor.spot_electricity_is_cheapest` - Single cheapest hour
- `binary_sensor.spot_electricity_is_cheapest_2_hours_block` - Cheapest 2-hour block
- `binary_sensor.spot_electricity_is_cheapest_3_hours_block` - Cheapest 3-hour block
- `binary_sensor.spot_electricity_is_cheapest_4_hours_block` - Cheapest 4-hour block
- `binary_sensor.spot_electricity_is_cheapest_6_hours_block` - Cheapest 6-hour block
- `binary_sensor.spot_electricity_is_cheapest_8_hours_block` - Cheapest 8-hour block
- `sensor.current_spot_electricity_price` - Current electricity price in CZK/kWh

### Solax Control Entities
- `select.solax_remotecontrol_power_control` - Battery control mode
- `number.solax_remotecontrol_duration` - Duration between repeats (seconds)
- `number.solax_remotecontrol_autorepeat_duration` - Total charging duration (seconds)
- `number.solax_remotecontrol_active_power` - Charging power (W)
- `button.solax_remotecontrol_trigger` - Start charging button

### Notification
- `notify.telegram` - Telegram notification service

## Operation Logic

### Start Conditions
1. Smart charging is enabled
2. Current month is Oct-Mar (seasonal mode)
3. Battery is not fully charged (<100%)
4. Energy deficit > 500 Wh
5. Currently in optimal cheap electricity window

### Stop Conditions
1. Battery reaches 100%, OR
2. Cheap electricity period ends

### Charging Parameters
- **Duration**: 20 seconds between autorepeat triggers
- **Autorepeat Duration**: Calculated based on energy needed (1-8 hours)
- **Active Power**: Optimized based on available time window

## Notifications

The system sends Telegram notifications:

**Start Notification:**
```
*SMART GRID CHARGE*
Smart charging started!

Battery: 45.2%
Deficit: 5600 Wh
Power: 2800 W
Est. time: 2.0h
Price: 3.50 CZK/kWh
```

**Stop Notification:**
```
*SMART CHARGE COMPLETE*
Smart charging stopped!

Final battery: 100.0%
Target reached: 100%
```

## Troubleshooting

### Common Issues

1. **Automation not triggering**
   - Check if `input_boolean.smart_charging_enabled` is ON
   - Verify current month is Oct-Mar
   - Confirm Czech Energy Spot Prices integration is working
   - Check if electricity price sensors are available

2. **Charging not stopping at 100%**
   - Verify battery sensors are reporting correct values
   - Check if Solax control entities are responding

3. **Wrong charging power**
   - Adjust `input_number.smart_charging_max_power`
   - Check battery deficit calculation

4. **Missing electricity price sensors**
   - Install and configure the Czech Energy Spot Prices integration
   - Verify integration is providing the required binary sensors

### Debugging

Enable automation traces in Home Assistant to see detailed execution logs. The templates include extensive comments for debugging calculations.

## Customization

### Modify Seasonal Behavior
Change the seasonal condition in both automations:
```yaml
{% set month = now().month %}
{{ month >= 10 or month <= 3 }}  # Oct-Mar active
```

### Adjust Minimum Energy Threshold
Change the 500 Wh threshold in the conditions:
```yaml
{{ energyDeficit > 500 and optimalBlock != 'none' and inTimeWindow }}
```

### Modify Time Limits
Adjust the autorepeat duration limits:
```yaml
{{ [secondsNeeded, 28800] | min | max(3600) }}  # 1-8 hours
```

## Version History

- **v1.0** - Initial release with smart time window selection and dynamic power adjustment

## Support

This automation system is designed to work with Solax inverter systems and requires the specific sensor setup described above. Modify entity names as needed for your configuration.

The Czech Energy Spot Prices integration is essential for providing the electricity price data and cheapest time period detection.
