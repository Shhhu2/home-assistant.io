---
title: "Battery Monitoring Automation"
description: "Automatically monitor all device batteries, alert on low levels, and receive periodic status reports."
related:
  - docs: /docs/automation/trigger/
    title: Automation Triggers
  - docs: /integrations/template/
    title: Template Integration
  - docs: /integrations/notify/
    title: Notifications
---

This example demonstrates how to create a comprehensive battery monitoring system that:

- Dynamically discovers all devices with batteries
- Alerts you when any battery drops to 25% or below
- Sends a twice-weekly summary of all battery statuses
- Identifies whether batteries need replacement or recharging

## Complete Configuration

Add the following to your {% term "`configuration.yaml`" %} file:

{% raw %}

```yaml
# Battery Monitoring Automation System
# =====================================
# This configuration provides comprehensive battery monitoring for all devices

# Template sensors for battery monitoring
template:
  - sensor:
      # Sensor that lists all battery entities and their states
      - name: "All Battery Devices"
        unique_id: all_battery_devices_sensor
        state: >
          {% set battery_entities = states.sensor
             | selectattr('attributes.device_class', 'defined')
             | selectattr('attributes.device_class', 'eq', 'battery')
             | list %}
          {{ battery_entities | count }}
        attributes:
          devices: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {% set ns = namespace(devices=[]) %}
            {% for entity in battery_entities %}
              {% set device_name = entity.name | replace(' Battery', '') | replace(' battery', '') %}
              {% set battery_level = entity.state | int(0) %}
              {% set battery_type = entity.attributes.get('battery_type', 'Unknown') %}
              {% if battery_type == 'Unknown' %}
                {% if 'rechargeable' in entity.entity_id or 'phone' in entity.entity_id or 'tablet' in entity.entity_id or 'watch' in entity.entity_id or 'laptop' in entity.entity_id %}
                  {% set battery_type = 'Rechargeable' %}
                {% else %}
                  {% set battery_type = 'Replaceable' %}
                {% endif %}
              {% endif %}
              {% set ns.devices = ns.devices + [{'name': device_name, 'entity_id': entity.entity_id, 'level': battery_level, 'type': battery_type}] %}
            {% endfor %}
            {{ ns.devices }}
          low_battery_devices: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {% set ns = namespace(devices=[]) %}
            {% for entity in battery_entities %}
              {% if entity.state | int(100) <= 25 %}
                {% set device_name = entity.name | replace(' Battery', '') | replace(' battery', '') %}
                {% set battery_level = entity.state | int(0) %}
                {% set battery_type = entity.attributes.get('battery_type', 'Unknown') %}
                {% if battery_type == 'Unknown' %}
                  {% if 'rechargeable' in entity.entity_id or 'phone' in entity.entity_id or 'tablet' in entity.entity_id or 'watch' in entity.entity_id or 'laptop' in entity.entity_id %}
                    {% set battery_type = 'Rechargeable' %}
                  {% else %}
                    {% set battery_type = 'Replaceable' %}
                  {% endif %}
                {% endif %}
                {% set ns.devices = ns.devices + [{'name': device_name, 'entity_id': entity.entity_id, 'level': battery_level, 'type': battery_type}] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}
          last_scan: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"

# Automations for battery monitoring
automation battery_monitor:
  # Automation 1: Check batteries every 20 minutes and alert on low levels
  - id: battery_low_level_check
    alias: "Battery Low Level Monitor"
    description: "Scans all battery devices every 20 minutes and sends alerts for batteries at 25% or below"
    mode: single
    triggers:
      - trigger: time_pattern
        minutes: "/20"
    actions:
      - variables:
          low_batteries: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {% set ns = namespace(devices=[]) %}
            {% for entity in battery_entities %}
              {% if entity.state | int(100) <= 25 %}
                {% set device_name = entity.name | replace(' Battery', '') | replace(' battery', '') %}
                {% set battery_level = entity.state | int(0) %}
                {% set battery_type = entity.attributes.get('battery_type', 'Unknown') %}
                {% if battery_type == 'Unknown' %}
                  {% if 'rechargeable' in entity.entity_id or 'phone' in entity.entity_id or 'tablet' in entity.entity_id or 'watch' in entity.entity_id or 'laptop' in entity.entity_id %}
                    {% set battery_type = 'Rechargeable' %}
                  {% else %}
                    {% set battery_type = 'Replaceable' %}
                  {% endif %}
                {% endif %}
                {% set action = 'recharge' if battery_type == 'Rechargeable' else 'replace' %}
                {% set ns.devices = ns.devices + [{'name': device_name, 'level': battery_level, 'type': battery_type, 'action': action}] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}
      - condition: template
        value_template: "{{ low_batteries | count > 0 }}"
      - action: notify.notify
        data:
          title: "🔋 Low Battery Alert"
          message: >
            {% for device in low_batteries %}
            ⚠️ {{ device.name }}
               Level: {{ device.level }}%
               Type: {{ device.type }}
               Action: Please {{ device.action }} the battery
            {% endfor %}

  # Automation 2: Twice weekly battery status report (Wednesday and Sunday at 9 AM)
  - id: battery_weekly_status_report
    alias: "Battery Weekly Status Report"
    description: "Sends a comprehensive battery status report twice a week"
    mode: single
    triggers:
      - trigger: time
        at: "09:00:00"
    conditions:
      - condition: time
        weekday:
          - wed
          - sun
    actions:
      - variables:
          all_batteries: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {% set ns = namespace(devices=[]) %}
            {% for entity in battery_entities %}
              {% set device_name = entity.name | replace(' Battery', '') | replace(' battery', '') %}
              {% set battery_level = entity.state | int(0) %}
              {% set battery_type = entity.attributes.get('battery_type', 'Unknown') %}
              {% if battery_type == 'Unknown' %}
                {% if 'rechargeable' in entity.entity_id or 'phone' in entity.entity_id or 'tablet' in entity.entity_id or 'watch' in entity.entity_id or 'laptop' in entity.entity_id %}
                  {% set battery_type = 'Rechargeable' %}
                {% else %}
                  {% set battery_type = 'Replaceable' %}
                {% endif %}
              {% endif %}
              {% set ns.devices = ns.devices + [{'name': device_name, 'level': battery_level, 'type': battery_type}] %}
            {% endfor %}
            {{ ns.devices | sort(attribute='level') }}
          critical_count: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {{ battery_entities | selectattr('state', 'lt', '25') | list | count }}
          low_count: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {{ battery_entities | selectattr('state', 'lt', '50') | selectattr('state', 'ge', '25') | list | count }}
      - action: notify.notify
        data:
          title: "🔋 Battery Status Report"
          message: >
            📊 Battery Status Summary
            ━━━━━━━━━━━━━━━━━━━━━━━━
            Total Devices: {{ all_batteries | count }}
            🔴 Critical (< 25%): {{ critical_count }}
            🟡 Low (25-50%): {{ low_count }}
            🟢 Good (> 50%): {{ all_batteries | count - (critical_count | int) - (low_count | int) }}

            📋 All Device Batteries:
            {% for device in all_batteries %}
            {% if device.level <= 25 %}🔴{% elif device.level <= 50 %}🟡{% else %}🟢{% endif %} {{ device.name }}: {{ device.level }}% ({{ device.type }})
            {% endfor %}

            Report generated: {{ now().strftime('%A, %B %d at %I:%M %p') }}

  # Automation 3: Immediate alert when a battery drops below 25%
  - id: battery_critical_drop_alert
    alias: "Battery Critical Drop Alert"
    description: "Sends an immediate alert when any battery crosses below the 25% threshold"
    mode: parallel
    max: 10
    triggers:
      - trigger: numeric_state
        entity_id: sensor.all_battery_devices
        attribute: low_battery_devices
        above: 0
    actions:
      - variables:
          battery_type: >
            {% set entity_id = trigger.entity_id %}
            {% if 'rechargeable' in entity_id or 'phone' in entity_id or 'tablet' in entity_id or 'watch' in entity_id or 'laptop' in entity_id %}
              Rechargeable
            {% else %}
              Replaceable
            {% endif %}
          action_needed: >
            {% if battery_type == 'Rechargeable' %}
              recharge
            {% else %}
              replace
            {% endif %}
      - action: notify.notify
        data:
          title: "⚠️ Battery Critical Alert"
          message: >
            A device battery has dropped below 25%!

            Check your devices and {{ action_needed }} batteries as needed.

            Use the weekly report or check the All Battery Devices sensor for details.
```

{% endraw %}

## Configuration Options

### Customizing the Low Battery Threshold

To change the 25% threshold, modify the comparison value in the templates. For example, to alert at 30%:

{% raw %}
```yaml
{% if entity.state | int(100) <= 30 %}
```
{% endraw %}

### Adjusting the Scan Interval

The default scan interval is 20 minutes. To change this, modify the `time_pattern` trigger:

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/30"  # Scan every 30 minutes
```

### Changing the Weekly Report Schedule

By default, reports are sent on Wednesday and Sunday at 9 AM. Modify the time and weekday conditions:

```yaml
triggers:
  - trigger: time
    at: "08:00:00"  # Change the time
conditions:
  - condition: time
    weekday:
      - mon  # Monday
      - thu  # Thursday
```

### Customizing Notifications

Replace `notify.notify` with your specific notification service:

```yaml
- action: notify.mobile_app_your_phone
  data:
    title: "🔋 Low Battery Alert"
    message: "..."
```

## Understanding Battery Types

The automation attempts to identify battery types:

- **Rechargeable**: Devices like phones, tablets, watches, laptops
- **Replaceable**: Devices like sensors, remotes, door locks

You can customize the detection logic by modifying the template conditions or by adding a `battery_type` attribute to your device entities using [customization](/docs/configuration/customizing-devices/).

## Adding Custom Battery Type Detection

To improve battery type detection for your specific devices, create a customize entry:

```yaml
homeassistant:
  customize:
    sensor.front_door_sensor_battery:
      battery_type: "CR2032"
    sensor.motion_sensor_battery:
      battery_type: "AA Rechargeable"
```

## Troubleshooting

### No batteries detected

Ensure your battery sensors have the `device_class: battery` attribute set. You can verify this in {% my developer_states title="Developer Tools > States" %}.

### Notifications not sending

1. Verify your notification service is configured correctly
2. Check that the service name matches exactly (e.g., `notify.mobile_app_your_phone`)
3. Review the automation trace in {% my automations title="Settings > Automations" %}

### Template errors

Use {% my developer_template title="Developer Tools > Template" %} to test your templates and debug any issues.
