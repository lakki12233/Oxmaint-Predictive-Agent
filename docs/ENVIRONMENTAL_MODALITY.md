# Environmental/Transactional Modality

## Overview
The environmental modality analyzes operational context and maintenance history to adjust failure risk predictions. This modality uses rule-based logic to assess how operating conditions affect asset health.

## Input Fields

| Field | Type | Description |
|-------|------|-------------|
| `operating_hours` | float | Total operating hours since last maintenance |
| `days_since_last_maintenance` | float | Days elapsed since last maintenance event |
| `ambient_temperature_c` | float | Ambient temperature in Celsius |
| `ambient_humidity_percent` | float | Ambient humidity percentage (0-100) |
| `load_factor` | float | Current load vs rated capacity (0-1) |
| `maintenance_overdue` | bool | Whether maintenance schedule is overdue |

## Risk Adjustment Logic

The environmental module computes a **risk multiplier** (0.5 to 2.0) based on:

### Operating Hours
- **High Risk** (≥2000h): 1.4x multiplier
- **Medium Risk** (≥1000h): 1.2x multiplier
- **Normal** (<1000h): 1.0x multiplier

### Maintenance Status
- **Critical Overdue** (≥60 days overdue): 1.5x multiplier
- **Overdue** (≥30 days or flagged): 1.3x multiplier
- **On Schedule**: 1.0x multiplier

### Temperature
- **High** (≥45°C): 1.15x multiplier
- **Low** (≤5°C): 1.1x multiplier
- **Normal**: 1.0x multiplier

### Humidity
- **High** (≥85%): 1.15x multiplier
- **Normal**: 1.0x multiplier

### Load Factor
- **Overloaded** (≥0.95): 1.2x multiplier
- **High** (≥0.80): 1.1x multiplier
- **Normal**: 1.0x multiplier

## Fusion with Other Modalities

The environmental risk multiplier is applied to the base failure probability computed from sensor data:

```
p_fail_final = min(1.0, p_fail_sensor × env_multiplier)
```

If environmental conditions are critical (multiplier ≥ 1.5) and no other fault is detected, the system assigns `"environmental_stress"` as the fault type.

## Example Scenarios

### Scenario 1: Well-Maintained Asset
```json
{
  "environmental": {
    "operating_hours": 450,
    "days_since_last_maintenance": 10,
    "ambient_temperature_c": 20,
    "ambient_humidity_percent": 45,
    "load_factor": 0.65,
    "maintenance_overdue": false
  }
}
```
**Effect**: Low risk multiplier (~1.0), favorable conditions

### Scenario 2: Critical Conditions
```json
{
  "environmental": {
    "operating_hours": 2500,
    "days_since_last_maintenance": 90,
    "ambient_temperature_c": 48,
    "ambient_humidity_percent": 88,
    "load_factor": 0.98,
    "maintenance_overdue": true
  }
}
```
**Effect**: High risk multiplier (~2.0), elevated failure probability

## Testing

Test environmental modality:
```powershell
# Test all scenarios
.\tests\scripts\test_environmental.ps1

# Or test individual files
$body = Get-Content .\samples\env_critical.json -Raw
Invoke-RestMethod -Uri "http://localhost:8000/predict" `
  -Method Post -ContentType "application/json" -Body $body
```

## Top Signals

Environmental signals appear in `top_signals` output:
- `env:conditions_critical` / `env:conditions_elevated` / `env:conditions_normal` / `env:conditions_favorable`
- `env:high_hours(2500h)` - Operating hours status
- `env:maint_critical(90d)` - Maintenance overdue status
- `env:temp_high(48.0C)` - Temperature alerts
- `env:humidity_high(88%)` - Humidity alerts
- `env:overloaded(0.98)` - Load factor status

## Implementation

See [`app/modalities/environmental.py`](../app/modalities/environmental.py) for the complete implementation.
