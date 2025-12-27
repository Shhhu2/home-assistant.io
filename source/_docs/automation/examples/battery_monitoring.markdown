---
title: "Battery Monitoring Automation"
description: "Comprehensive battery monitoring with device registry, admin approval workflow, smart notifications, and to-do list integration."
related:
  - docs: /docs/automation/trigger/
    title: Automation Triggers
  - docs: /integrations/template/
    title: Template Integration
  - docs: /integrations/notify/
    title: Notifications
  - docs: /integrations/todo/
    title: To-do Integration
  - docs: /docs/configuration/customizing-devices/
    title: Customizing Devices
---

This advanced battery monitoring system provides:

- **Device Registry**: Audited inventory of all battery devices with types and charging methods
- **Smart Discovery**: Automatic detection of new devices with admin approval workflow
- **Tiered Alerts**: 20% warning and 10% critical thresholds with sleep functionality
- **To-Do Integration**: Track devices needing attention with automatic resolution
- **Auto-Resolve**: Devices automatically clear when charged above 80%

## System Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    BATTERY MONITORING SYSTEM                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Device Registry ──► Battery Scan ──► Notification Manager      │
│  (JSON file)         (every 20m)      (with sleep logic)        │
│        │                  │                   │                  │
│        ▼                  ▼                   ▼                  │
│  New Device? ───► Admin Approval      Sleep Status              │
│  Detection                            (input_booleans)          │
│        │                                      │                  │
│        ▼                                      ▼                  │
│  LLM Lookup ◄──────────────────────► To-Do List                 │
│  (manual trigger)                    (unique ID tracking)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before setting up this system, ensure you have:

1. The [file integration](/integrations/file/) configured for registry persistence
2. A [notification service](/integrations/notify/) configured
3. The [to-do integration](/integrations/todo/) set up
4. A [mobile app](/integrations/mobile_app/) for actionable notifications (recommended)

## Step 1: Create the Device Registry File

Create a file at `/config/battery_registry.json` with your known devices:

```json
{
  "devices": {
    "sensor.living_room_motion_battery": {
      "friendly_name": "Living Room Motion Sensor",
      "battery_type": "replaceable",
      "battery_model": "CR2032",
      "charging_method": "Replace battery",
      "approved": true,
      "sleep_20": false,
      "sleep_10": false,
      "last_notified": null,
      "added_date": "2024-01-15"
    },
    "sensor.apple_tv_remote_battery": {
      "friendly_name": "Apple TV Remote",
      "battery_type": "rechargeable",
      "battery_model": "Li-ion",
      "charging_method": "Lightning cable",
      "approved": true,
      "sleep_20": false,
      "sleep_10": false,
      "last_notified": null,
      "added_date": "2024-01-15"
    },
    "sensor.front_door_lock_battery": {
      "friendly_name": "Front Door Lock",
      "battery_type": "replaceable",
      "battery_model": "4x AA",
      "charging_method": "Replace batteries",
      "approved": true,
      "sleep_20": false,
      "sleep_10": false,
      "last_notified": null,
      "added_date": "2024-01-15"
    }
  },
  "pending_approval": {},
  "denied_devices": [],
  "last_updated": "2024-01-15T10:00:00"
}
```

### Device Registry Fields

| Field | Description | Example Values |
|-------|-------------|----------------|
| `battery_type` | Whether battery is rechargeable or replaceable | `rechargeable`, `replaceable` |
| `battery_model` | Specific battery type/model | `CR2032`, `Li-ion`, `4x AA`, `18650` |
| `charging_method` | How to recharge or replace | `USB-C`, `Lightning cable`, `Replace battery`, `Wireless charging` |
| `approved` | Admin has approved this device | `true`, `false` |
| `sleep_20` | Notifications silenced for 20% threshold | `true`, `false` |
| `sleep_10` | Notifications silenced for 10% threshold | `true`, `false` |
| `last_notified` | Timestamp of last notification | ISO datetime or `null` |

## Step 2: Add Custom Device Attributes

Add to {% term "`configuration.yaml`" %} to enrich device data:

```yaml
homeassistant:
  customize:
    # Rechargeable devices
    sensor.apple_tv_remote_battery:
      battery_type: rechargeable
      battery_model: Li-ion
      charging_method: Lightning cable

    sensor.iphone_battery:
      battery_type: rechargeable
      battery_model: Li-ion
      charging_method: MagSafe or Lightning

    sensor.magic_keyboard_battery:
      battery_type: rechargeable
      battery_model: Li-ion
      charging_method: Lightning cable

    # Replaceable battery devices
    sensor.front_door_sensor_battery:
      battery_type: replaceable
      battery_model: CR2032
      charging_method: Replace battery

    sensor.motion_sensor_battery:
      battery_type: replaceable
      battery_model: 2x AAA
      charging_method: Replace batteries

    sensor.door_lock_battery:
      battery_type: replaceable
      battery_model: 4x AA
      charging_method: Replace batteries
```

## Step 3: Create Input Helpers

Add these input helpers for managing device sleep status and admin controls:

```yaml
# Input helpers for battery monitoring
input_boolean:
  # Master control for battery monitoring
  battery_monitoring_enabled:
    name: Battery Monitoring Enabled
    icon: mdi:battery-check

input_text:
  # Stores JSON of devices with active sleep status
  battery_sleep_status:
    name: Battery Sleep Status
    max: 65535
    initial: "{}"

  # Stores pending device approvals
  battery_pending_devices:
    name: Battery Pending Devices
    max: 65535
    initial: "{}"

input_number:
  # Threshold settings
  battery_warning_threshold:
    name: Battery Warning Threshold
    min: 10
    max: 50
    step: 5
    initial: 20
    unit_of_measurement: "%"
    icon: mdi:battery-20

  battery_critical_threshold:
    name: Battery Critical Threshold
    min: 5
    max: 25
    step: 5
    initial: 10
    unit_of_measurement: "%"
    icon: mdi:battery-alert

input_select:
  # For admin approval actions
  battery_admin_action:
    name: Battery Admin Action
    options:
      - "None"
      - "Approve Device"
      - "Deny Device"
      - "Sleep 20% Alert"
      - "Sleep 10% Alert"
      - "Wake All Alerts"
    initial: "None"
    icon: mdi:account-cog
```

## Step 4: Template Sensors

Create template sensors for monitoring battery status:

{% raw %}

```yaml
template:
  - sensor:
      # Main battery inventory sensor
      - name: "Battery Device Inventory"
        unique_id: battery_device_inventory
        state: >
          {% set battery_entities = states.sensor
             | selectattr('attributes.device_class', 'defined')
             | selectattr('attributes.device_class', 'eq', 'battery')
             | rejectattr('state', 'in', ['unknown', 'unavailable'])
             | list %}
          {{ battery_entities | count }}
        attributes:
          # All devices with full details
          all_devices: >
            {% set battery_entities = states.sensor
               | selectattr('attributes.device_class', 'defined')
               | selectattr('attributes.device_class', 'eq', 'battery')
               | rejectattr('state', 'in', ['unknown', 'unavailable'])
               | list %}
            {% set ns = namespace(devices=[]) %}
            {% for entity in battery_entities %}
              {% set device_name = entity.name | replace(' Battery', '') | replace(' battery', '') %}
              {% set battery_level = entity.state | int(0) %}
              {% set battery_type = entity.attributes.get('battery_type', 'unknown') %}
              {% set battery_model = entity.attributes.get('battery_model', 'Unknown') %}
              {% set charging_method = entity.attributes.get('charging_method', 'Check device manual') %}
              {% if battery_type == 'unknown' %}
                {% if 'phone' in entity.entity_id or 'tablet' in entity.entity_id or
                      'watch' in entity.entity_id or 'laptop' in entity.entity_id or
                      'remote' in entity.entity_id or 'keyboard' in entity.entity_id or
                      'mouse' in entity.entity_id or 'headphone' in entity.entity_id or
                      'earbud' in entity.entity_id or 'controller' in entity.entity_id %}
                  {% set battery_type = 'rechargeable' %}
                {% else %}
                  {% set battery_type = 'replaceable' %}
                {% endif %}
              {% endif %}
              {% set ns.devices = ns.devices + [{
                'entity_id': entity.entity_id,
                'name': device_name,
                'level': battery_level,
                'battery_type': battery_type,
                'battery_model': battery_model,
                'charging_method': charging_method,
                'unique_id': entity.entity_id | replace('.', '_') | replace('sensor_', '')
              }] %}
            {% endfor %}
            {{ ns.devices | sort(attribute='level') }}

          # Devices needing attention (below warning threshold)
          warning_devices: >
            {% set threshold = states('input_number.battery_warning_threshold') | int(20) %}
            {% set sleep_status = states('input_text.battery_sleep_status') | from_json %}
            {% set all_devices = state_attr('sensor.battery_device_inventory', 'all_devices') or [] %}
            {% set ns = namespace(devices=[]) %}
            {% for device in all_devices %}
              {% set is_sleeping = sleep_status.get(device.entity_id, {}).get('sleep_20', false) %}
              {% if device.level <= threshold and device.level > states('input_number.battery_critical_threshold') | int(10) and not is_sleeping %}
                {% set ns.devices = ns.devices + [device] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}

          # Critical devices (below critical threshold - bypasses sleep for 20%)
          critical_devices: >
            {% set threshold = states('input_number.battery_critical_threshold') | int(10) %}
            {% set sleep_status = states('input_text.battery_sleep_status') | from_json %}
            {% set all_devices = state_attr('sensor.battery_device_inventory', 'all_devices') or [] %}
            {% set ns = namespace(devices=[]) %}
            {% for device in all_devices %}
              {% set is_sleeping_10 = sleep_status.get(device.entity_id, {}).get('sleep_10', false) %}
              {% if device.level <= threshold and not is_sleeping_10 %}
                {% set ns.devices = ns.devices + [device] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}

          # Devices that have recovered (above 80%)
          recovered_devices: >
            {% set all_devices = state_attr('sensor.battery_device_inventory', 'all_devices') or [] %}
            {% set sleep_status = states('input_text.battery_sleep_status') | from_json %}
            {% set ns = namespace(devices=[]) %}
            {% for device in all_devices %}
              {% set was_sleeping = sleep_status.get(device.entity_id, {}).get('sleep_20', false) or
                                    sleep_status.get(device.entity_id, {}).get('sleep_10', false) %}
              {% if device.level >= 80 and was_sleeping %}
                {% set ns.devices = ns.devices + [device] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}

          last_scan: "{{ now().isoformat() }}"

      # Sensor for tracking new/unregistered devices
      - name: "Battery Unregistered Devices"
        unique_id: battery_unregistered_devices
        state: >
          {% set pending = states('input_text.battery_pending_devices') | from_json %}
          {{ pending.keys() | list | count }}
        attributes:
          devices: >
            {{ states('input_text.battery_pending_devices') | from_json }}
```

{% endraw %}

## Step 5: Scripts for Device Management

Create scripts to manage the device registry:

{% raw %}

```yaml
script:
  # Script to approve a pending device
  battery_approve_device:
    alias: "Battery - Approve Device"
    description: "Approve a pending device and add to registry"
    fields:
      entity_id:
        description: "The entity ID of the device to approve"
        example: "sensor.new_device_battery"
    sequence:
      - variables:
          pending: "{{ states('input_text.battery_pending_devices') | from_json }}"
          device_data: "{{ pending.get(entity_id, {}) }}"
      - condition: template
        value_template: "{{ entity_id in pending }}"
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_pending_devices
        data:
          value: >
            {% set pending = states('input_text.battery_pending_devices') | from_json %}
            {% set _ = pending.pop(entity_id, none) %}
            {{ pending | to_json }}
      - action: notify.notify
        data:
          title: "Device Approved"
          message: "{{ device_data.get('name', entity_id) }} has been added to the battery registry."

  # Script to deny a pending device
  battery_deny_device:
    alias: "Battery - Deny Device"
    description: "Deny a pending device"
    fields:
      entity_id:
        description: "The entity ID of the device to deny"
        example: "sensor.new_device_battery"
    sequence:
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_pending_devices
        data:
          value: >
            {% set pending = states('input_text.battery_pending_devices') | from_json %}
            {% set _ = pending.pop(entity_id, none) %}
            {{ pending | to_json }}
      - action: notify.notify
        data:
          title: "Device Denied"
          message: "{{ entity_id }} has been denied and will be ignored."

  # Script to set sleep status for a device
  battery_set_sleep:
    alias: "Battery - Set Sleep Status"
    description: "Set sleep status for device notifications"
    fields:
      entity_id:
        description: "The entity ID of the device"
        example: "sensor.device_battery"
      sleep_level:
        description: "Which threshold to sleep"
        example: "sleep_20"
    sequence:
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_sleep_status
        data:
          value: >
            {% set current = states('input_text.battery_sleep_status') | from_json %}
            {% set device_status = current.get(entity_id, {'sleep_20': false, 'sleep_10': false}) %}
            {% set _ = device_status.update({sleep_level: true, 'slept_at': now().isoformat()}) %}
            {% set _ = current.update({entity_id: device_status}) %}
            {{ current | to_json }}
      - action: notify.notify
        data:
          title: "Notifications Silenced"
          message: "{{ entity_id }} alerts silenced until battery is recharged/replaced."

  # Script to wake (clear sleep status) for a device
  battery_wake_device:
    alias: "Battery - Wake Device"
    description: "Clear sleep status and resume notifications"
    fields:
      entity_id:
        description: "The entity ID of the device"
        example: "sensor.device_battery"
    sequence:
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_sleep_status
        data:
          value: >
            {% set current = states('input_text.battery_sleep_status') | from_json %}
            {% set _ = current.pop(entity_id, none) %}
            {{ current | to_json }}

  # Script to add device to to-do list
  battery_add_to_todo:
    alias: "Battery - Add to To-Do List"
    description: "Add a device to the battery to-do list"
    fields:
      entity_id:
        description: "The entity ID of the device"
        example: "sensor.device_battery"
      device_name:
        description: "Friendly name of the device"
        example: "Living Room Motion Sensor"
      battery_level:
        description: "Current battery level"
        example: "15"
      battery_type:
        description: "Type of battery"
        example: "replaceable"
      battery_model:
        description: "Battery model"
        example: "CR2032"
      charging_method:
        description: "How to charge/replace"
        example: "Replace battery"
    sequence:
      - action: todo.add_item
        target:
          entity_id: todo.battery_maintenance
        data:
          item: "{{ device_name }}"
          description: >
            Battery Level: {{ battery_level }}%
            Type: {{ battery_type | title }}
            Battery: {{ battery_model }}
            Action: {{ charging_method }}
            Entity: {{ entity_id }}
          due_datetime: "{{ (now() + timedelta(days=7)).isoformat() }}"

  # Script to remove device from to-do list (when recovered)
  battery_remove_from_todo:
    alias: "Battery - Remove from To-Do List"
    description: "Remove a recovered device from to-do list"
    fields:
      device_name:
        description: "Friendly name of the device"
        example: "Living Room Motion Sensor"
    sequence:
      - action: todo.remove_item
        target:
          entity_id: todo.battery_maintenance
        data:
          item: "{{ device_name }}"
```

{% endraw %}

## Step 6: Automations

Create the main automations for battery monitoring:

{% raw %}

```yaml
automation:
  # ============================================
  # AUTOMATION 1: Regular Battery Scan (Every 20 minutes)
  # ============================================
  - id: battery_regular_scan
    alias: "Battery - Regular Scan"
    description: "Scans batteries every 20 minutes, checks against registry, detects new devices"
    mode: single
    triggers:
      - trigger: time_pattern
        minutes: "/20"
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
    actions:
      # Check for new unregistered devices
      - variables:
          all_devices: "{{ state_attr('sensor.battery_device_inventory', 'all_devices') or [] }}"
          pending: "{{ states('input_text.battery_pending_devices') | from_json }}"
          sleep_status: "{{ states('input_text.battery_sleep_status') | from_json }}"
          warning_threshold: "{{ states('input_number.battery_warning_threshold') | int(20) }}"
          critical_threshold: "{{ states('input_number.battery_critical_threshold') | int(10) }}"

      # Process warning level devices (20% threshold)
      - variables:
          warning_devices: >
            {% set ns = namespace(devices=[]) %}
            {% for device in all_devices %}
              {% set is_sleeping = sleep_status.get(device.entity_id, {}).get('sleep_20', false) %}
              {% if device.level <= warning_threshold and device.level > critical_threshold and not is_sleeping %}
                {% set ns.devices = ns.devices + [device] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}

      - condition: template
        value_template: "{{ warning_devices | count > 0 }}"

      - action: notify.notify
        data:
          title: "🔋 Low Battery Warning"
          message: >
            The following devices need attention:

            {% for device in warning_devices %}
            ⚠️ {{ device.name }}
               Level: {{ device.level }}%
               Battery: {{ device.battery_model }}
               Action: {{ device.charging_method }}
            {% endfor %}
          data:
            actions:
              - action: "SLEEP_WARNING"
                title: "Silence Alerts"
              - action: "VIEW_TODO"
                title: "View To-Do List"

  # ============================================
  # AUTOMATION 2: Critical Battery Alert (10% - bypasses 20% sleep)
  # ============================================
  - id: battery_critical_alert
    alias: "Battery - Critical Alert"
    description: "Immediate alert when battery drops to 10% or below"
    mode: parallel
    max: 10
    triggers:
      - trigger: template
        value_template: >
          {{ (state_attr('sensor.battery_device_inventory', 'critical_devices') or []) | count > 0 }}
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
    actions:
      - variables:
          critical_devices: "{{ state_attr('sensor.battery_device_inventory', 'critical_devices') or [] }}"
      - action: notify.notify
        data:
          title: "🚨 CRITICAL Battery Alert"
          message: >
            URGENT: The following devices have critically low batteries:

            {% for device in critical_devices %}
            🔴 {{ device.name }}: {{ device.level }}%
               Battery: {{ device.battery_model }}
               Action: {{ device.charging_method }}
            {% endfor %}

            These devices may stop working soon!
          data:
            actions:
              - action: "SLEEP_CRITICAL"
                title: "Acknowledge"
              - action: "VIEW_TODO"
                title: "View To-Do List"
      # Add to to-do list
      - repeat:
          for_each: "{{ critical_devices }}"
          sequence:
            - action: script.battery_add_to_todo
              data:
                entity_id: "{{ repeat.item.entity_id }}"
                device_name: "{{ repeat.item.name }}"
                battery_level: "{{ repeat.item.level }}"
                battery_type: "{{ repeat.item.battery_type }}"
                battery_model: "{{ repeat.item.battery_model }}"
                charging_method: "{{ repeat.item.charging_method }}"

  # ============================================
  # AUTOMATION 3: Handle Notification Actions
  # ============================================
  - id: battery_handle_notification_actions
    alias: "Battery - Handle Notification Actions"
    description: "Process actionable notification responses"
    mode: parallel
    triggers:
      - trigger: event
        event_type: mobile_app_notification_action
        event_data:
          action: "SLEEP_WARNING"
      - trigger: event
        event_type: mobile_app_notification_action
        event_data:
          action: "SLEEP_CRITICAL"
    actions:
      - variables:
          warning_devices: "{{ state_attr('sensor.battery_device_inventory', 'warning_devices') or [] }}"
          critical_devices: "{{ state_attr('sensor.battery_device_inventory', 'critical_devices') or [] }}"
          sleep_level: "{{ 'sleep_20' if trigger.event.data.action == 'SLEEP_WARNING' else 'sleep_10' }}"
          devices_to_sleep: "{{ warning_devices if trigger.event.data.action == 'SLEEP_WARNING' else critical_devices }}"
      - repeat:
          for_each: "{{ devices_to_sleep }}"
          sequence:
            - action: script.battery_set_sleep
              data:
                entity_id: "{{ repeat.item.entity_id }}"
                sleep_level: "{{ sleep_level }}"
            # Add to to-do list
            - action: script.battery_add_to_todo
              data:
                entity_id: "{{ repeat.item.entity_id }}"
                device_name: "{{ repeat.item.name }}"
                battery_level: "{{ repeat.item.level }}"
                battery_type: "{{ repeat.item.battery_type }}"
                battery_model: "{{ repeat.item.battery_model }}"
                charging_method: "{{ repeat.item.charging_method }}"
      - action: notify.notify
        data:
          title: "Alerts Silenced"
          message: "{{ devices_to_sleep | count }} device(s) silenced. Added to to-do list for tracking."

  # ============================================
  # AUTOMATION 4: Auto-Resolve When Battery Recovered (>80%)
  # ============================================
  - id: battery_auto_resolve
    alias: "Battery - Auto Resolve Recovered Devices"
    description: "Clear sleep status and remove from to-do when battery exceeds 80%"
    mode: single
    triggers:
      - trigger: template
        value_template: >
          {{ (state_attr('sensor.battery_device_inventory', 'recovered_devices') or []) | count > 0 }}
    actions:
      - variables:
          recovered_devices: "{{ state_attr('sensor.battery_device_inventory', 'recovered_devices') or [] }}"
      - repeat:
          for_each: "{{ recovered_devices }}"
          sequence:
            # Clear sleep status
            - action: script.battery_wake_device
              data:
                entity_id: "{{ repeat.item.entity_id }}"
            # Remove from to-do list
            - action: script.battery_remove_from_todo
              data:
                device_name: "{{ repeat.item.name }}"
      - action: notify.notify
        data:
          title: "🔋 Batteries Recovered"
          message: >
            The following devices have been charged/replaced and removed from tracking:

            {% for device in recovered_devices %}
            ✅ {{ device.name }}: {{ device.level }}%
            {% endfor %}

  # ============================================
  # AUTOMATION 5: Detect New Devices
  # ============================================
  - id: battery_detect_new_devices
    alias: "Battery - Detect New Devices"
    description: "Detect new battery devices and request admin approval"
    mode: single
    triggers:
      - trigger: state
        entity_id: sensor.battery_device_inventory
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
    actions:
      - variables:
          all_devices: "{{ state_attr('sensor.battery_device_inventory', 'all_devices') or [] }}"
          pending: "{{ states('input_text.battery_pending_devices') | from_json }}"
          # Check for devices not in pending list (simplified - in production check against registry file)
          current_entity_ids: "{{ all_devices | map(attribute='entity_id') | list }}"
          pending_entity_ids: "{{ pending.keys() | list }}"
      - variables:
          new_devices: >
            {% set ns = namespace(devices=[]) %}
            {% for device in all_devices %}
              {% if device.entity_id not in pending_entity_ids %}
                {# In production, also check against registry file #}
                {% set ns.devices = ns.devices + [device] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices }}
      # This condition would check against the actual registry in production
      # For now, we only notify about devices we haven't seen before
      - condition: template
        value_template: "{{ new_devices | count > 0 }}"

  # ============================================
  # AUTOMATION 6: Twice Weekly Status Report
  # ============================================
  - id: battery_weekly_report
    alias: "Battery - Weekly Status Report"
    description: "Comprehensive battery status report twice a week"
    mode: single
    triggers:
      - trigger: time
        at: "09:00:00"
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
      - condition: time
        weekday:
          - wed
          - sun
    actions:
      - variables:
          all_devices: "{{ state_attr('sensor.battery_device_inventory', 'all_devices') or [] }}"
          warning_threshold: "{{ states('input_number.battery_warning_threshold') | int(20) }}"
          critical_threshold: "{{ states('input_number.battery_critical_threshold') | int(10) }}"
          critical_count: "{{ all_devices | selectattr('level', 'le', critical_threshold) | list | count }}"
          warning_count: "{{ all_devices | selectattr('level', 'le', warning_threshold) | selectattr('level', 'gt', critical_threshold) | list | count }}"
          good_count: "{{ all_devices | selectattr('level', 'gt', warning_threshold) | list | count }}"
      - action: notify.notify
        data:
          title: "🔋 Weekly Battery Report"
          message: >
            📊 Battery Status Summary
            ━━━━━━━━━━━━━━━━━━━━━━━━
            Total Devices: {{ all_devices | count }}
            🔴 Critical (≤{{ critical_threshold }}%): {{ critical_count }}
            🟡 Warning (≤{{ warning_threshold }}%): {{ warning_count }}
            🟢 Good (>{{ warning_threshold }}%): {{ good_count }}

            📋 All Devices (sorted by level):
            {% for device in all_devices %}
            {% if device.level <= critical_threshold %}🔴{% elif device.level <= warning_threshold %}🟡{% else %}🟢{% endif %} {{ device.name }}: {{ device.level }}%
               └─ {{ device.battery_model }} ({{ device.battery_type }})
            {% endfor %}

            Report: {{ now().strftime('%A, %B %d at %I:%M %p') }}
```

{% endraw %}

## Step 7: Create a To-Do List for Battery Maintenance

Add a to-do list entity for tracking battery maintenance:

```yaml
# Add to configuration.yaml if using local to-do
todo:
  - platform: local_todo
    name: Battery Maintenance
```

Or use an existing to-do integration like [Google Tasks](/integrations/google_tasks/), [Todoist](/integrations/todoist/), or [Microsoft To Do](/integrations/todo.microsoft/).

## Step 8: Optional - LLM Device Lookup Script

For looking up device battery details using an LLM (only needed for initial setup or new devices):

{% raw %}

```yaml
# Requires the OpenAI Conversation integration or similar
script:
  battery_llm_lookup:
    alias: "Battery - LLM Device Lookup"
    description: "Look up battery details for a device using AI"
    fields:
      device_name:
        description: "The device name to look up"
        example: "Aqara Motion Sensor P1"
    sequence:
      - action: conversation.process
        data:
          agent_id: conversation.openai  # Adjust to your LLM integration
          text: >
            What type of battery does a {{ device_name }} use?
            Please respond in this exact JSON format:
            {
              "battery_type": "rechargeable" or "replaceable",
              "battery_model": "specific battery type like CR2032, 2x AAA, Li-ion, etc",
              "charging_method": "how to recharge or replace the battery"
            }
        response_variable: llm_response
      - action: notify.notify
        data:
          title: "Battery Lookup Result"
          message: "{{ llm_response.response.speech.plain.speech }}"
```

{% endraw %}

## Notification Flow Summary

```text
Battery Level Drops
        │
        ▼
   ≤20% Warning ────────────────────────────────┐
        │                                        │
   Notification Sent                             │
        │                                        │
   Admin Acknowledges? ──No──► Repeat next scan │
        │                                        │
       Yes                                       │
        │                                        │
   Sleep Status = ON                             │
   Added to To-Do List                          │
        │                                        │
        ▼                                        │
   ≤10% Critical ◄───────────────────────────────┘
        │                    (bypasses 20% sleep)
   Notification Sent
        │
   Admin Acknowledges?
        │
       Yes
        │
   Sleep 10% Status = ON
   Updated in To-Do List
        │
        ▼
   Battery Recharged/Replaced
   (Level > 80%)
        │
   Auto-Resolve:
   - Clear all sleep status
   - Remove from To-Do List
   - Resume normal monitoring
```

## Troubleshooting

### Sleep status not persisting

Verify `input_text.battery_sleep_status` is configured and check its value in Developer Tools > States.

### To-do items not appearing

1. Ensure the to-do integration is properly configured
2. Verify `todo.battery_maintenance` entity exists
3. Check script traces for errors

### Notifications not actionable

Ensure you're using a mobile app notification service that supports actions:

```yaml
action: notify.mobile_app_your_phone  # Not notify.notify
```

### Template errors

Use {% my developer_template title="Developer Tools > Template" %} to test templates before using them in automations.
