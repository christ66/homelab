# Gaming Power Normalized Metrics

This package maps Home Assistant TP-Link/Kasa power-strip entities into a device-agnostic Prometheus schema through the `smart-power-exporter` service.

Detected Home Assistant strips:

- `strip-1`: `TP-LINK_Power Strip_03E7`, six metered plugs.
- `strip-2`: `TP-LINK_Power Strip_FB48`, six metered plugs.
- `strip-3`: `TP-LINK_Power Strip_A2E0`, six metered plugs.

The Home Assistant token is intentionally not stored in this public Git repository. Create or rotate the live Kubernetes Secret named `smart-power-home-assistant-token` in the `monitoring` namespace.

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

## Live exporter

The `smart-power-exporter` Deployment reads Home Assistant through the in-cluster service URL:

`http://home-assistant.home-assistant.svc.cluster.local:8123`

It exposes normalized metrics on:

`http://smart-power-exporter.monitoring.svc.cluster.local:9108/metrics`

Create the Secret before deploying or reconciling the exporter:

```sh
kubectl -n monitoring create secret generic smart-power-home-assistant-token --from-literal=token='<home-assistant-token>'
```

Use the actual strip ratings before relying on alert severity. The current config uses conservative planning defaults of 120 V and 12 A continuous capacity per detected strip until you replace them with the real device ratings.

Because the three strips are plugged into wall outlets that may share one branch circuit, combined-load alerting is intentionally based on one shared circuit budget: `shared_circuit_rated_volts: 120` and `shared_circuit_rated_continuous_amps: 12`. That yields combined warning at `1152 W` and combined critical at `1296 W`. Per-strip thresholds remain independent.
