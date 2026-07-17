# Gaming Power Normalized Metrics

This package did not detect an existing homelab repository or monitoring stack in the mounted workspace, so it ships a device-agnostic normalized schema. Map your real smart-plug or smart-strip telemetry into these metric names before enabling alerts.

## Required labels

Use exactly three strip identifiers:

- `strip="strip-1"`
- `strip="strip-2"`
- `strip="strip-3"`

Every outlet-level metric must also include a stable `outlet` label such as `outlet="outlet-1"` or `outlet="pc"`.

## Measurement metrics

| Metric | Labels | Unit | Required |
| --- | --- | --- | --- |
| `smart_power_outlet_watts` | `strip`, `outlet` | W | yes |
| `smart_power_outlet_amps` | `strip`, `outlet` | A | when available |
| `smart_power_outlet_voltage_volts` | `strip`, `outlet` | V | when available |
| `smart_power_outlet_energy_kwh` | `strip`, `outlet` | kWh | when available |
| `smart_power_outlet_on` | `strip`, `outlet` | `1` on, `0` off | when available |
| `smart_power_outlet_last_seen_timestamp_seconds` | `strip`, `outlet` | Unix seconds | yes |

The recording rules derive strip and combined totals from outlet metrics so the dashboard works consistently across exporters.

## Threshold metrics

Thresholds are metrics so ratings remain configurable and can vary per strip and outlet:

| Metric | Labels | Unit |
| --- | --- | --- |
| `smart_power_outlet_warning_watts` | `strip`, `outlet` | W |
| `smart_power_outlet_critical_watts` | `strip`, `outlet` | W |
| `smart_power_strip_warning_watts` | `strip` | W |
| `smart_power_strip_critical_watts` | `strip` | W |
| `smart_power_combined_warning_watts` | none | W |
| `smart_power_combined_critical_watts` | none | W |

Calculate warning at 80% of rated continuous capacity and critical at 90% of rated continuous capacity. For common US 120 V / 15 A strips, continuous-load planning commonly uses 12 A / 1440 W, but do not assume your strips share that rating. Use the actual voltage, rated amps, outlet rating, and any manufacturer continuous-load guidance for each device.

## Mapping examples

Do not copy these blindly; metric names vary by integration.

- Home Assistant Prometheus integration: create recording rules from entity metrics for each outlet sensor and switch state.
- Shelly exporter: map each channel's power, voltage, current, energy, relay state, and update timestamp into the normalized outlet metrics.
- Kasa exporter: map per-plug or per-child outlet values into the normalized outlet metrics.
- Tasmota or MQTT: bridge MQTT telemetry into Prometheus with labels for strip and outlet, then emit the normalized metrics directly.
- UniFi or other PDU exporters: prefer native outlet/channel labels when available, then relabel to `strip` and `outlet`.

The key rule is that raw device metrics stay outside the dashboard and alerts. Convert them once into the normalized schema, then let the recording rules and dashboard consume only `smart_power_*` and `gaming_power:*` series.
