---
title: openSenseMap
description: Instructions on how to setup openSenseMap sensors in Home Assistant.
ha_category:
  - Health
ha_release: 0.85
ha_iot_class: Cloud Polling
ha_config_flow: true
ha_domain: opensensemap
ha_platforms:
  - air_quality
  - sensor
ha_integration_type: service
related:
  - docs: /docs/configuration/
    title: Configuration file
ha_quality_scale: legacy
---

The **openSenseMap** {% term integration %} queries the open data API of [openSenseMap.org](https://opensensemap.org/) to monitor an air-quality sensor station.

## Setup

To find the ID of a station, open it on [openSenseMap](https://opensensemap.org/) and copy the last segment of the URL — for example, `5b450e565dc1ec001bf7cd1d` in [https://opensensemap.org/explore/5b450e565dc1ec001bf7cd1d](https://opensensemap.org/explore/5b450e565dc1ec001bf7cd1d).

{% include integrations/config_flow.md %}

{% configuration_basic %}
Station ID:
  description: The ID of the openSenseMap station to monitor.
{% endconfiguration_basic %}

## Sensors

Each configured station is exposed as a device with the following sensor entities. Sensors are only populated when the configured station reports the corresponding measurement.

Enabled by default:

- **PM2.5** — particulate matter under 2.5 µm (µg/m³)
- **PM10** — particulate matter under 10 µm (µg/m³)
- **Temperature** (°C)
- **Humidity** (%)
- **Atmospheric pressure** (hPa)

Disabled by default — enable them from the entity registry if your station reports these measurements:

- **PM1** — particulate matter under 1 µm (µg/m³)
- **Illuminance** (lx)
- **UV index**
- **Wind speed** (m/s)
- **Wind direction** (°)
- **Precipitation** (mm)

## YAML configuration is deprecated

Previously, openSenseMap was configured under the `air_quality` platform in {% term "`configuration.yaml`" %}:

```yaml
# Example configuration.yaml entry (deprecated)
air_quality:
  - platform: opensensemap
    station_id: STATION_ID
```

This configuration method is **deprecated**. Existing YAML configuration is imported automatically the first time Home Assistant starts after the upgrade, and a repair issue is raised to remind you to remove the `air_quality` entry from your {% term "`configuration.yaml`" %} file. Support for YAML configuration will be removed in a future release.
