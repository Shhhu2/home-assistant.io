---
title: "Battery Monitoring Automation"
description: "Smart battery monitoring with notification sleep, to-do tracking, and auto-recovery using Battery Notes integration."
related:
  - docs: /docs/automation/trigger/
    title: Automation Triggers
  - docs: /integrations/todo/
    title: To-do Integration
  - url: https://github.com/andrew-codechimp/HA-Battery-Notes
    title: Battery Notes Integration
---

This battery monitoring system leverages the [Battery Notes](https://github.com/andrew-codechimp/HA-Battery-Notes) integration for device discovery and adds smart notification management on top.

## What This System Does

- **Alerts** when batteries drop below thresholds (20% warning, 10% critical)
- **Sleeps** notifications per-device after you acknowledge them
- **Tracks** low battery devices in your to-do list
- **Auto-resolves** when batteries are recharged/replaced (>80%)
- **Detects** devices that stop communicating (unavailable)
- **Reports** weekly battery status summary

## Architecture

```text
┌──────────────────────────────────────────────────────────────┐
│                     BATTERY NOTES (HACS)                      │
│  • Auto-discovers all battery devices                         │
│  • Library of 1000+ device battery types                      │
│  • Tracks battery levels and changes                          │
│  • Detects devices not reporting                              │
└──────────────────────┬───────────────────────────────────────┘
                       │ Events
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    YOUR AUTOMATIONS                           │
│  • Low Battery Alert (with sleep check)                       │
│  • Auto-Recovery (clear sleep, remove from to-do)             │
│  • Weekly Report                                              │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    TO-DO LIST                                 │
│  • Devices needing attention                                  │
│  • Auto-removed when recovered                                │
└──────────────────────────────────────────────────────────────┘
```

## Prerequisites

1. Install [Battery Notes](https://github.com/andrew-codechimp/HA-Battery-Notes) via HACS
2. Configure a [notification service](/integrations/notify/)
3. Set up the [to-do integration](/integrations/todo/) (optional but recommended)

## Step 1: Install Battery Notes

Install via HACS:

1. Open HACS in Home Assistant
2. Click **Integrations**
3. Search for "Battery Notes"
4. Install and restart Home Assistant
5. Go to {% my integrations title="Settings > Devices & Services" %}
6. Add the Battery Notes integration

Battery Notes will automatically discover your battery devices and provide:
- Battery type and model information
- Events when batteries cross thresholds
- Events when devices stop reporting
- Events when battery levels increase (replaced/recharged)

## Step 2: Create Input Helpers

Add these minimal helpers to your {% term "`configuration.yaml`" %}:

```yaml
input_boolean:
  battery_monitoring_enabled:
    name: Battery Monitoring
    icon: mdi:battery-check

input_text:
  # Stores sleep status as JSON: {"sensor.device_battery": {"level": "20", "slept_at": "..."}}
  battery_sleep_status:
    name: Battery Sleep Status
    max: 65535
    initial: "{}"
```

## Step 3: Create the Automations

Add these three automations to handle all battery monitoring:

{% raw %}

```yaml
automation:
  # ============================================
  # AUTOMATION 1: Low Battery Alert
  # Triggers on Battery Notes threshold events
  # ============================================
  - id: battery_low_alert
    alias: "Battery - Low Alert"
    description: "Alert when battery drops below threshold, respecting sleep status"
    mode: queued
    max: 20
    triggers:
      # Battery Notes fires this event when battery crosses threshold
      - trigger: event
        event_type: battery_notes_battery_threshold
        event_data:
          battery_low: true
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
    actions:
      - variables:
          device_id: "{{ trigger.event.data.device_id }}"
          device_name: "{{ trigger.event.data.device_name }}"
          battery_level: "{{ trigger.event.data.battery_level }}"
          battery_type: "{{ trigger.event.data.battery_type | default('Unknown') }}"
          battery_type_and_quantity: "{{ trigger.event.data.battery_type_and_quantity | default('Check device') }}"
          sleep_status: "{{ states('input_text.battery_sleep_status') | from_json }}"
          is_critical: "{{ battery_level | int <= 10 }}"
          is_sleeping: "{{ sleep_status.get(device_id, {}).get('level', '') == '20' and not is_critical }}"

      # Skip if sleeping at 20% level (critical 10% bypasses sleep)
      - condition: template
        value_template: "{{ not is_sleeping }}"

      # Send notification
      - action: notify.notify
        data:
          title: "{{ '🚨 CRITICAL' if is_critical else '🔋 Low' }} Battery: {{ device_name }}"
          message: |
            {{ device_name }}: {{ battery_level }}%
            Battery: {{ battery_type_and_quantity }}
            {{ 'Needs immediate attention!' if is_critical else 'Consider replacing/recharging soon.' }}
          data:
            actions:
              - action: "BATTERY_SLEEP_{{ device_id }}"
                title: "{{ 'Acknowledge' if is_critical else 'Silence Alerts' }}"
              - action: "BATTERY_VIEW_TODO"
                title: "View To-Do"
            tag: "battery_{{ device_id }}"

      # Add to to-do list
      - action: todo.add_item
        target:
          entity_id: todo.battery_maintenance
        data:
          item: "{{ device_name }}"
          description: |
            Battery: {{ battery_level }}%
            Type: {{ battery_type_and_quantity }}
            Device ID: {{ device_id }}

  # ============================================
  # AUTOMATION 2: Handle Sleep Action
  # Sets sleep status when user acknowledges
  # ============================================
  - id: battery_handle_sleep
    alias: "Battery - Handle Sleep Action"
    description: "Set sleep status when user acknowledges notification"
    mode: queued
    triggers:
      - trigger: event
        event_type: mobile_app_notification_action
    conditions:
      - condition: template
        value_template: "{{ trigger.event.data.action.startswith('BATTERY_SLEEP_') }}"
    actions:
      - variables:
          device_id: "{{ trigger.event.data.action.replace('BATTERY_SLEEP_', '') }}"
          current_status: "{{ states('input_text.battery_sleep_status') | from_json }}"
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_sleep_status
        data:
          value: >
            {% set updated = current_status.copy() %}
            {% set _ = updated.update({device_id: {'level': '20', 'slept_at': now().isoformat()}}) %}
            {{ updated | to_json }}
      - action: notify.notify
        data:
          message: "Alerts silenced for this device until battery is replaced/recharged."

  # ============================================
  # AUTOMATION 3: Auto-Recovery
  # Clears sleep and to-do when battery recovers
  # ============================================
  - id: battery_auto_recovery
    alias: "Battery - Auto Recovery"
    description: "Clear sleep status and to-do when battery exceeds 80%"
    mode: queued
    max: 20
    triggers:
      # Battery Notes fires this when battery level increases
      - trigger: event
        event_type: battery_notes_battery_increased
    conditions:
      - condition: template
        value_template: "{{ trigger.event.data.battery_level | int >= 80 }}"
    actions:
      - variables:
          device_id: "{{ trigger.event.data.device_id }}"
          device_name: "{{ trigger.event.data.device_name }}"
          battery_level: "{{ trigger.event.data.battery_level }}"
          current_status: "{{ states('input_text.battery_sleep_status') | from_json }}"
          was_sleeping: "{{ device_id in current_status }}"

      # Only notify if device was being tracked
      - condition: template
        value_template: "{{ was_sleeping }}"

      # Clear sleep status
      - action: input_text.set_value
        target:
          entity_id: input_text.battery_sleep_status
        data:
          value: >
            {% set updated = current_status.copy() %}
            {% set _ = updated.pop(device_id, none) %}
            {{ updated | to_json }}

      # Remove from to-do
      - action: todo.remove_item
        target:
          entity_id: todo.battery_maintenance
        data:
          item: "{{ device_name }}"

      # Notify recovery
      - action: notify.notify
        data:
          title: "✅ Battery Recovered"
          message: "{{ device_name }} is now at {{ battery_level }}%. Removed from tracking."
          data:
            tag: "battery_{{ device_id }}"

  # ============================================
  # AUTOMATION 4: Device Not Reporting Alert
  # Catches devices that go silent
  # ============================================
  - id: battery_not_reporting
    alias: "Battery - Device Not Reporting"
    description: "Alert when a battery device stops communicating"
    mode: queued
    triggers:
      - trigger: event
        event_type: battery_notes_battery_not_reported
    conditions:
      - condition: state
        entity_id: input_boolean.battery_monitoring_enabled
        state: "on"
    actions:
      - variables:
          device_name: "{{ trigger.event.data.device_name }}"
          last_reported: "{{ trigger.event.data.last_reported }}"
      - action: notify.notify
        data:
          title: "⚠️ Device Not Reporting"
          message: |
            {{ device_name }} hasn't reported since {{ last_reported }}.
            The device may be offline or the battery may be dead.

  # ============================================
  # AUTOMATION 5: Weekly Report
  # Summary of all battery statuses
  # ============================================
  - id: battery_weekly_report
    alias: "Battery - Weekly Report"
    description: "Send weekly battery status summary"
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
          batteries: >
            {% set ns = namespace(devices=[]) %}
            {% for state in states.sensor %}
              {% if state.attributes.device_class is defined and state.attributes.device_class == 'battery' %}
                {% if state.state not in ['unknown', 'unavailable'] %}
                  {% set ns.devices = ns.devices + [{
                    'name': state.name | replace(' Battery', ''),
                    'level': state.state | int(0)
                  }] %}
                {% endif %}
              {% endif %}
            {% endfor %}
            {{ ns.devices | sort(attribute='level') }}
          critical: "{{ batteries | selectattr('level', 'le', 10) | list }}"
          warning: "{{ batteries | selectattr('level', 'le', 20) | rejectattr('level', 'le', 10) | list }}"
          good: "{{ batteries | selectattr('level', 'gt', 20) | list }}"
      - action: notify.notify
        data:
          title: "🔋 Weekly Battery Report"
          message: |
            📊 Summary
            Total: {{ batteries | length }} devices
            🔴 Critical (≤10%): {{ critical | length }}
            🟡 Warning (≤20%): {{ warning | length }}
            🟢 Good (>20%): {{ good | length }}

            {% if critical | length > 0 %}
            🔴 Critical:
            {% for d in critical %}• {{ d.name }}: {{ d.level }}%
            {% endfor %}{% endif %}
            {% if warning | length > 0 %}
            🟡 Warning:
            {% for d in warning %}• {{ d.name }}: {{ d.level }}%
            {% endfor %}{% endif %}
```

{% endraw %}

## Step 4: Create To-Do List (Optional)

If you want to track batteries in a to-do list:

```yaml
todo:
  - platform: local_todo
    name: Battery Maintenance
```

Or connect an existing service like [Google Tasks](/integrations/google_tasks/), [Todoist](/integrations/todoist/), or [Microsoft To Do](/integrations/todo.microsoft/).

## How It Works

### Notification Flow

```text
Battery drops below 20%
        │
        ▼
Battery Notes fires "battery_notes_battery_threshold" event
        │
        ▼
Our automation checks sleep status
        │
   ┌────┴────┐
   │         │
Sleeping   Not Sleeping
   │         │
   │         ▼
   │    Send notification + Add to to-do
   │         │
   │         ▼
   │    User taps "Silence Alerts"
   │         │
   │         ▼
   │    Sleep status = ON
   │         │
   └────┬────┘
        │
        ▼
Battery drops below 10% (CRITICAL)
        │
        ▼
BYPASSES sleep → Always notifies
        │
        ▼
User taps "Acknowledge"
        │
        ▼
Sleep status updated for critical level
        │
        ▼
Battery recharged/replaced (>80%)
        │
        ▼
Battery Notes fires "battery_notes_battery_increased" event
        │
        ▼
Our automation clears sleep + removes from to-do
        │
        ▼
Notification: "Battery Recovered"
```

### Sleep Logic Explained

| Battery Level | Sleep at 20%? | Result |
|---------------|---------------|--------|
| 18% (first drop) | No | Notification sent |
| 18% (acknowledged) | Yes | No more notifications |
| 15% | Yes | Still sleeping, no notification |
| 9% (critical) | Yes, but bypassed | Notification sent (critical bypasses 20% sleep) |
| 85% (recovered) | N/A | Sleep cleared, removed from to-do |

## Configuration Options

### Adjust Thresholds

Battery Notes has its own threshold settings. Configure them in the integration options:

1. Go to {% my integrations title="Settings > Devices & Services" %}
2. Find Battery Notes
3. Click **Configure**
4. Set your desired thresholds

### Change Notification Service

Replace `notify.notify` with your specific service:

```yaml
- action: notify.mobile_app_your_phone
```

### Change Weekly Report Schedule

Modify the time and days:

```yaml
triggers:
  - trigger: time
    at: "08:00:00"  # Different time
conditions:
  - condition: time
    weekday:
      - mon  # Different days
      - fri
```

## Comparison: Before vs After Simplification

| Aspect | Before (Complex) | After (Simplified) |
|--------|------------------|-------------------|
| **Integrations** | None (custom) | Battery Notes (HACS) |
| **Automations** | 6 | 5 |
| **Scripts** | 5 | 0 (inline actions) |
| **Input Helpers** | 6 | 2 |
| **Template Sensors** | 2 complex | 0 (use Battery Notes) |
| **JSON Registry File** | Required | Not needed |
| **LLM Lookup** | Custom script | Not needed (library) |
| **Device Discovery** | Custom templates | Automatic |
| **Lines of YAML** | ~800 | ~200 |
| **Setup Time** | 30+ minutes | 10 minutes |

## Troubleshooting

### Battery Notes events not firing

1. Verify Battery Notes is installed and configured
2. Check that devices have `device_class: battery`
3. Review Battery Notes settings for threshold values

### Sleep status not persisting

Check `input_text.battery_sleep_status` value in Developer Tools > States

### To-do items not appearing

1. Ensure `todo.battery_maintenance` entity exists
2. If using a different to-do service, update the entity_id in automations

### Notifications not actionable

Use a mobile app notification service that supports actions:

```yaml
action: notify.mobile_app_your_phone  # Not notify.notify
```

## Advanced: Custom Thresholds Per Device

If you need different thresholds for specific devices, use Battery Notes' per-device configuration or add a condition:

{% raw %}

```yaml
# Example: Alert at 30% for critical devices
- condition: template
  value_template: >
    {% set critical_devices = ['front_door_lock', 'smoke_detector'] %}
    {% set threshold = 30 if device_id in critical_devices else 20 %}
    {{ battery_level | int <= threshold }}
```

{% endraw %}
