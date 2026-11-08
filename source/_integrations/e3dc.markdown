---
title: E3/DC
description: Instructions on how to integrate an E3/DC home power station with Home Assistant over Modbus TCP.
ha_categories:
  - energy.solar_batteries
ha_release: "2026.12"
ha_iot_class: Local Polling
ha_codeowners:
  - '@AlCalzone'
ha_domain: e3dc
ha_config_flow: true
ha_integration_type: device
ha_platforms:
  - number
  - sensor
  - switch
ha_quality_scale: bronze
ha_zeroconf: true
---

The **E3/DC** {% term integration %} connects Home Assistant to an E3/DC home power station from HagerEnergy over Modbus TCP. It reads the power flows between solar, battery, house, grid and wallboxes, and can control the battery and the grid feed-in.

## Use cases

- Seeing where your power goes. Solar production, battery charging and discharging, house consumption and grid import and export are all live sensors.
- Running appliances on surplus solar. Start a heavy load once the solar power exceeds what the house uses.
- Charging the battery from the grid while power is cheap, or capping the feed-in when the grid operator asks for it.
- Knowing when the house runs on backup power. The emergency power sensor shows when the power station supplies the house during a grid failure.

## Supported devices

The integration supports E3/DC home power stations whose Modbus interface uses the E3/DC protocol ("Simple Mode"), such as the S10 E, S10 E PRO, S10 X, S20 X PRO and E3/DC one, and the systems running the Q10 and P10 firmware lines.

## Unsupported devices

The SunSpec protocol mode of the Modbus interface is not supported. E3/DC systems reached only over the RSCP protocol are not supported either.

## Prerequisites

1. On the power station, go to **Main menu** > **Smart Functions** > **Smart Home** > **Modbus**.
2. Turn **Modbus** on, and set **Protocol** to **E3DC**.
3. Note the port. The default is `502`.

The power station answers Modbus requests only from devices in its own subnet.

{% include integrations/config_flow.md %}

{% configuration_basic %}
Host:
  description: "The hostname or IP address of the power station."
Port:
  description: "The Modbus TCP port set in the power station's Smart Home menu. The default is `502`."
{% endconfiguration_basic %}

A discovered power station is set up on the default port `502`. If you set a different port, add the integration manually.

{% note %}
Home Assistant can share a Modbus connection between integrations. Another integration that already talks to the same power station shares the connection instead of opening a second one.
{% endnote %}

## Supported functionality

### Sensors

The power station device has these sensors:

- **PV power**, **Battery power**, **House power** and **Grid power**, in watts. Battery power is positive while discharging and negative while charging. Grid power is positive while importing and negative while feeding in.
- **Battery**: the battery's state of charge.
- **Self-sufficiency** and **Self-consumption**, in percent.

These sensors depend on the installed hardware:

- **Wallbox power** and **Wallbox solar power**, in watts, if a wallbox is configured.
- **Additional feeder power**, in watts, if a power meter measures an additional feeder, such as a second PV inverter.
- **Emergency power**, if the power station supports emergency power. It shows whether emergency power is active, inactive or not available. On the S10 E and S10 E PRO, **Motor switch off** means the emergency power motor switch is not in its on position.

The integration checks the wallboxes and power meters when it starts. After you add a wallbox or a power meter, reload the integration.

### Numbers and switches for remote control

On firmware that supports remote control, the power station device has three setpoints. Each is a number and a switch:

- **Battery power setpoint** and **Battery power control**: charge (positive) or discharge (negative) the battery at a fixed power. PV power charges first, and the grid makes up the rest.
- **Feed-in limit setpoint** and **Feed-in limit control**: cap the power fed into the grid.
- **Active power limit setpoint** and **Active power limit control**: cap the inverter's output, as a percentage of its rated power. The cap ignores the house's consumption.

Setting a number writes the value to the power station and turns its switch on. The power station drops a setpoint 30 seconds after the last write, so while the switch is on, the integration writes the value again every 10 seconds. Turning the switch off hands control back: remote battery control ends at once, the integration writes 100 % as the active power limit, and the feed-in limit lapses within 30 seconds. Writing 100 % is itself a setpoint, so once the active power failsafe timeout runs out, the active power failsafe value applies. Check on your power station that this is what you want. Turning the switch on again holds the last value set, which is kept across restarts.

The numbers and switches stay usable while a poll fails, because holding a setpoint doesn't depend on the last poll. A switch shows unknown while writing to the power station fails, because the power station may have dropped the setpoint. If a setpoint spans two registers and can only be half-written, setting it fails and the error says what the integration tried to put back.

## Examples

### Charge the battery from the grid while power is cheap

This automation charges at 3 kW while a price sensor is below 15 cents, and hands the battery back when the price rises.

{% raw %}

```yaml
alias: "Charge the battery while power is cheap"
triggers:
  - trigger: numeric_state
    entity_id: sensor.electricity_price
    below: 0.15
    id: cheap
  - trigger: numeric_state
    entity_id: sensor.electricity_price
    above: 0.15
    id: expensive
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: cheap
        sequence:
          - action: number.set_value
            target:
              entity_id: number.e3_dc_s10_x_battery_power_setpoint
            data:
              value: 3000
      - conditions:
          - condition: trigger
            id: expensive
        sequence:
          - action: switch.turn_off
            target:
              entity_id: switch.e3_dc_s10_x_battery_power_control
```

{% endraw %}

## Data updates

The integration polls the power flows every 5 seconds.

## Known limitations

- A setpoint is held while Home Assistant runs. When Home Assistant stops, or when you reload or disable the integration, it hands every setpoint back, giving up after about 5 seconds on an unresponsive power station. If that fails, the switch shows off anyway. The battery power and the feed-in limit lapse within 30 seconds. The active power limit lasts for the active power failsafe timeout, after which the active power failsafe value applies. With a failsafe timeout of 0, it lasts until it is written again. After a restart, turn the setpoint's switch on again.

## Troubleshooting

{% details "Setup reports that the device is not an E3/DC power station" %}

### Symptom: "The device answered, but it is not an E3/DC power station in E3/DC protocol mode"

When you add the integration, the form shows this error. An integration that was already set up stops with "The device is not an E3/DC power station in E3/DC protocol mode."

#### Description

The integration checks register 40001 for the E3/DC identifier. The power station reports it only while its Modbus interface uses the E3/DC protocol.

#### Resolution

1. On the power station, go to **Main menu** > **Smart Functions** > **Smart Home** > **Modbus**.
2. Set **Protocol** to **E3DC**, not **SUN_SPEC**.
3. Add the integration again, or reload it.

{% enddetails %}

{% details "Setup fails to connect" %}

### Symptom: "Failed to connect"

When you add the integration, the form shows this error. An integration that was already set up keeps retrying, and its sensors are unavailable.

#### Description

Home Assistant got no answer from the power station at the host and port you entered.

#### Resolution

1. Check that Modbus is turned on in the power station's **Smart Home** menu, and that the port matches.
2. Check that Home Assistant is in the same subnet as the power station. The power station answers Modbus requests only from devices in its own subnet.
3. Check that the host is the power station's current IP address. To keep it from changing, give the power station a fixed IP address in your router.

{% enddetails %}

{% details "Setup reports a different serial number" %}

### Symptom: "The device at this address has serial number …, not …"

An integration that worked before stops with this error.

#### Description

Home Assistant recognizes your power station by the serial number it read when you added the integration. A different power station now answers at the host and port of this entry. For example, your router gave the IP address to another E3/DC power station.

#### Resolution

1. Give the power station of this entry its old IP address back, for example with a fixed IP address in your router.
2. To use the power station that answers now, delete this entry and add the integration again.

{% enddetails %}

{% details "Remote control numbers and switches are missing" %}

### Symptom: The remote control numbers and switches are missing

The power station device has no numbers or switches for the battery power, the feed-in limit or the active power limit.

#### Description

The firmware does not serve the remote control registers.

#### Resolution

Update the power station's firmware, then reload the integration.

{% enddetails %}

## Removing the integration

This integration follows standard integration removal.

{% include integrations/remove_device_service.md %}

After removing the integration, turn Modbus off in the power station's **Smart Home** menu if nothing else uses it.
