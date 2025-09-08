
# Device Registry: Developer Guide

This documentation explains **how to define a complete device profile** JSON, using the Device Registry schema for Device Kit projects. It covers required and optional fields, conventions, and best practices.  

**Audience:**  
- Device firmware integrators  
- Backend or app developers extending hardware support  
- QA validating device integration

## Table of Contents

1. [Device Registry Overview](#device-registry-overview)
2. [Schema Version & Structure](#schema-version--structure)
3. [Top-Level Required Fields](#top-level-required-fields)
4. [Device Profile Definition](#device-profile-definition)
    - [Identity (USB & BLE Matching)](#identity-usb--ble-matching)
    - [Transports](#transports)
    - [Services (BLE & USB)](#services-ble--usb)
    - [Components, Capabilities & Operations](#components-capabilities--operations)
    - [Settings & Groups](#settings--groups)
    - [Firmware](#firmware)
    - [Telemetry](#telemetry)
    - [Rules (Orchestration)](#rules-orchestration)
    - [Extensions](#extensions)
5. [Example Device Profile](#example-device-profile)
6. [Validation & Testing](#validation--testing)
7. [Best Practices & FAQ](#best-practices--faq)

---

## Device Registry Overview

A **Device Registry** is a JSON file describing all supported hardware device profiles for the app.  
Each profile encodes how to **discover**, **connect**, **interact**, and **configure** a physical device, and enables **rules-based orchestration** and **settings UI**—**with no extra code**.

---

## Schema Version & Structure

Every registry starts with these fields:

```json
{
  "schema": "https://dsync.app/schemas/device-registry/1-0-0",
  "version": "1.0.0",
  "devices": [
    // ... DeviceProfile objects
  ]
}
```

- **`schema`**: must match the canonical URL, do not change.
- **`version`**: semantic versioning of your registry (e.g. `"1.2.0"`).
- **`devices`**: array of one or more **DeviceProfile** objects (see below).

---

## Top-Level Required Fields

| Field     | Type    | Description                              | Required |
|-----------|---------|------------------------------------------|----------|
| `schema`  | String  | Schema version URL (fixed)               | Yes      |
| `version` | String  | Your registry version (`semver`)         | Yes      |
| `devices` | Array   | List of DeviceProfile objects            | Yes      |

---

## Device Profile Definition

Each entry in `devices[]` is a **DeviceProfile**.  
Below is the anatomy of a profile.

### Required DeviceProfile Fields

| Field           | Type         | Description                                              |
|-----------------|--------------|----------------------------------------------------------|
| `id`            | String       | Unique device profile ID (reverse-DNS recommended)        |
| `name`          | String/Object| Display name (supports localization)                      |
| `category`      | String       | Category (`pump`, `scale`, etc.)                          |
| `manufacturer`  | String       | Manufacturer name                                         |
| `identity`      | Object       | USB/BLE identity matching (at least one required)         |

#### Example:
```json
{
  "id": "com.dsync.diesel-pump.pro",
  "name": { "en": "Diesel Dispenser Pro", "af": "Diesel Dispenseerder Pro" },
  "category": "pump",
  "manufacturer": "FarmTrace",
  "identity": { ... }
  // more fields below
}
```

---

### Identity (USB & BLE Matching)

Used for device *discovery* and *matching* at runtime.

```json
"identity": {
  "usb": [
    { "vid": "0x2B24", "pid": "0x0112" }
  ],
  "ble": [
    {
      "localNamePrefix": "FarmTrace",
      "serviceUuidsAny": [
        "12345678-0000-1000-8000-00805f9b34fb"
      ]
    }
  ],
  "fallbackHint": "prefer-ble"
}
```

- **`usb`**: array of USB VID/PID matchers.  
- **`ble`**: array of BLE matchers (localNamePrefix, any service UUID, optional manufacturerId).
- **`fallbackHint`**: how to prioritize matching (`match-first`/`prefer-ble`/`prefer-usb`).  
- At least one of `usb` or `ble` is **required**.

---

### Transports

Describes transport configuration and protocol for USB/BLE.

```json
"transports": {
  "usb": {
    "protocol": "cdc-acm",
    "baud": 115200,
    "timeoutMs": 2000
  },
  "ble": {
    "mtu": 256,
    "connectionPriority": "high",
    "reconnect": { "maxAttempts": 5, "backoffMs": 3000 }
  }
}
```

**USB fields:**
- `protocol`: `cdc-acm`, `hid`, `vendor`, `custom`
- `baud`, `timeoutMs`, `endpoints`, etc.

**BLE fields:**
- `mtu`: Preferred MTU size
- `connectionPriority`: `high`, `balanced`, `lowPower`
- `reconnect`: policy for reconnection attempts

---

### Services (BLE & USB)

Defines *logical* service layers for your device.

#### BLE Example

```json
"services": {
  "ble": [
    {
      "uuid": "12345678-0000-1000-8000-00805f9b34fb",
      "label": "Main Service",
      "characteristics": [
        {
          "uuid": "23456789-0000-1000-8000-00805f9b34fb",
          "label": "State",
          "properties": ["read", "notify"],
          "format": "bytes"
        }
      ]
    }
  ]
}
```

#### USB Example

```json
"services": {
  "usb": [
    {
      "id": "main",
      "label": "Main Channel",
      "frame": { "type": "line", "delimiter": "" }
    }
  ]
}
```

**BLE:**
- Each service has `uuid`, `label`, `characteristics[]`.
- Characteristics specify: `uuid`, `properties` (read/write/notify), `format` (`bytes`, `struct`, `bitmask`).

**USB:**
- Each service has `id`, `label`, `frame` (type: `line` or `raw`).

---

### Components, Capabilities & Operations

**Components** represent logical units or controls (e.g., a motor, pump, sensor).

```json
"components": [
  {
    "id": "pump",
    "type": "pump",
    "label": "Pump Unit",
    "capabilities": [
      { "name": "flowRate", "unit": "L/min", "precision": 0.1 }
    ],
    "operations": {
      "start": {
        "transport": "ble",
        "serviceUuid": "12345678-0000-1000-8000-00805f9b34fb",
        "characteristicUuid": "34567890-0000-1000-8000-00805f9b34fb",
        "mode": "write",
        "writeHex": "01"
      },
      "stop": {
        "transport": "usb",
        "channelId": "main",
        "mode": "write",
        "write": "STOP"
      }
    }
  }
]
```

- **`capabilities`**: Declare measurable or actionable features.
- **`operations`**:  
    - Keyed by operation name.
    - Each operation specifies:  
        - `transport` (`usb`/`ble`)  
        - For BLE: `serviceUuid`, `characteristicUuid`, `mode`, `writeHex` or `write`  
        - For USB: `channelId`, `mode`, `writeHex` or `write`  
    - For **writes**, you must provide either `writeHex` or `write`.

---

### Settings & Groups

Defines UI-driven settings for configuration.  
Each setting is grouped, and supports types: `boolean`, `number`, `select`, `text`, `password`.

```json
"settings": {
  "groups": [
    {
      "id": "safety",
      "label": "Safety",
      "items": [
        {
          "key": "autoShutoffSeconds",
          "type": "number",
          "label": "Auto Shutoff Time (sec)",
          "min": 5,
          "max": 600,
          "default": 120,
          "step": 1,
          "persistence": "device"
        },
        {
          "key": "requirePin",
          "type": "boolean",
          "label": "Require PIN to start",
          "default": false
        }
      ]
    }
  ],
  "visibilityRules": [
    {
      "when": { "key": "requirePin", "equals": true },
      "show": ["safety.pinCode"]
    }
  ]
}
```

- **`groups[]`**: Each group contains `items[]`.
    - Each item: `key`, `type`, `label`, `default`, `min`, `max`, `step`, `options` (for select), `persistence` (`app`, `device`, `secure-app`).
- **`visibilityRules[]`**: Show/hide fields based on current settings.

---

### Firmware

Describe minimum required firmware and update URLs for OTA.

```json
"firmware": {
  "minVersion": "1.4.0",
  "updateUrl": "https://devices.mycompany.com/fw/device123.bin",
  "notes": "Required for multi-stage dispense"
}
```

---

### Telemetry

Describes which data is exported for monitoring/statistics.

```json
"telemetry": {
  "privacy": "minimal",
  "fields": [
    { "name": "totalDispensedL", "type": "counter" },
    { "name": "tempC", "type": "gauge" }
  ]
}
```

---

### Rules (Orchestration)

Rules enable **automatic actions** (e.g., shut off pump if flow = 0, or raise an alert).

```json
"rules": [
  {
    "when": { "cmp": { "left": "settings.safety.autoShutoffSeconds", "op": "<=", "right": 60 } },
    "then": [
      { "alert": "Warning: Short shutoff time!", "level": "warn" }
    ]
  }
]
```

- **`when`**: Condition (comparison, AND, OR, IN, etc.).
- **`then`**: Array of actions (`invoke`, `set`, `alert`). Actions can target other devices or settings.

---

### Extensions

Free-form, non-validated fields for vendor/app-specific extensions (must start with `x-`).

```json
"extensions": {
  "x-calibrationCurve": [1.0, 1.1, 1.2]
}
```

---

## Example Device Profile

Below is a simplified but valid device profile.  
**You should expand it according to your device’s capabilities and protocol.**

```json
{
  "id": "com.example.smart-pump",
  "name": "Smart Water Pump",
  "category": "pump",
  "manufacturer": "AquaTech",
  "identity": {
    "usb": [{ "vid": "0x1234", "pid": "0x5678" }],
    "ble": [{ "localNamePrefix": "AquaPump", "serviceUuidsAny": ["d502d000-8577-4d4c-a234-3f285bde35e8"] }]
  },
  "transports": {
    "usb": { "protocol": "cdc-acm", "baud": 9600, "timeoutMs": 1000 },
    "ble": { "mtu": 180, "connectionPriority": "high" }
  },
  "services": {
    "ble": [
      {
        "uuid": "d502d000-8577-4d4c-a234-3f285bde35e8",
        "label": "Pump Service",
        "characteristics": [
          {
            "uuid": "d502d001-8577-4d4c-a234-3f285bde35e8",
            "label": "Pump State",
            "properties": ["read", "notify"],
            "format": "bytes"
          }
        ]
      }
    ],
    "usb": [
      {
        "id": "console",
        "label": "Console Channel",
        "frame": { "type": "line", "delimiter": "" }
      }
    ]
  },
  "components": [
    {
      "id": "mainPump",
      "type": "pump",
      "label": "Pump",
      "operations": {
        "start": {
          "transport": "ble",
          "serviceUuid": "d502d000-8577-4d4c-a234-3f285bde35e8",
          "characteristicUuid": "d502d002-8577-4d4c-a234-3f285bde35e8",
          "mode": "write",
          "writeHex": "01"
        },
        "stop": {
          "transport": "usb",
          "channelId": "console",
          "mode": "write",
          "write": "STOP"
        }
      }
    }
  ],
  "settings": {
    "groups": [
      {
        "id": "main",
        "label": "Main Settings",
        "items": [
          {
            "key": "maxRuntime",
            "type": "number",
            "label": "Max Runtime (minutes)",
            "min": 1,
            "max": 180,
            "default": 60,
            "step": 1,
            "persistence": "device"
          }
        ]
      }
    ]
  },
  "firmware": { "minVersion": "1.0.0" }
}
```

---

## Validation & Testing

- Use the [RegistryLoader](#) class in the codebase to parse and validate your registry JSON before shipping.
- The loader enforces:
    - Unique device IDs and valid patterns.
    - At least one USB/BLE matcher per profile.
    - All required fields per operation.
    - Correct UUIDs, hex, and option values.

**Runtime:**  
If you load the registry in your app (e.g., as an asset), all matching, connection, settings, and operation wiring are automatic.

---

## Best Practices & FAQ

### How do I add localization?
- Any `label` or `name` field can be a string or `{ "en": "English", "af": "Afrikaans", ... }`.

### What if I have multiple hardware models?
- Use the `models` array to enumerate part numbers/variants.
- Use `tags` for grouping/filtering devices in UI or rules.

### What if two devices share the same BLE/USB UUIDs?
- Ensure the `identity` is specific enough (use manufacturerId, localNamePrefix, or both).
- Avoid overly broad BLE service UUIDs that match unrelated devices.

### What if my BLE device has no advertised service UUIDs?
- Use `localNamePrefix` and/or `manufacturerId` as fallbacks.

### How do I define a setting that should only appear when another setting is enabled?
- Use `visibilityRules` with the appropriate condition and `show` fields.

### How do I expose device telemetry?
- Populate the `telemetry` section; types include `counter`, `gauge`, and `histogram`.

### How do I orchestrate actions across devices?
- Define `rules` using conditions (`when`) and actions (`then`). Actions can target operations, set settings, or show alerts.

---

## Resources

- [Device-Registery-Spec.json](https://github.com/FarmtraceDev/Device-Kit-Specification/blob/main/Device-Registery-Spec.json) – Full JSON schema  
- [Example device profiles](https://github.com/FarmtraceDev/Device-Kit-Specification/blob/main/Device-Registery-Example.json)

# End of Guide

If you need a quick starter template for your device, just ask!
