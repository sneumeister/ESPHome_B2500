---
name: B2500 Netto Power/Current MQTT
overview: Berechnung und Publikation von Netto-Akku-Leistung und -Strom mit eindeutigen Topics ('netto') plus HA-Autodiscovery.
todos: []
isProject: false
---

# Plan: Netto-Leistung/-Strom für B2500 per MQTT + HA-Discovery

## Ziel
- Netto-Akku-Leistung: `P_netto = P_out_total - P_in_total` (Entladen > 0, Laden < 0) unter eigenem Topic mit "netto".
- Netto-Akku-Strom: `I_netto = P_netto / V_pack` (V_pack aus `cell_voltage.sum`), gleicher Vorzeichenkonvention.
- MQTT-State-Topics eindeutig, plus HA-Autodiscovery-Configs.

## Dateien/Orte
- ESPHome-Konfig: [d:\Projekte\ESPHome_B2500\config.yaml](d:\Projekte\ESPHome_B2500\config.yaml)

## Schritte
1) **Pack-Spannung als Template-Sensor**
   - Template-Sensor `battery_voltage_pack` (V) aus `b2500_device_cell_voltage_1.state` → JSON `sum` (Fallback: NaN bei Fehler).
2) **Netto-Leistung & -Strom berechnen**
   - Template-Sensor `battery_power_netto` = `out_total_power - total_power_in` (W, measurement, + Entladen).
   - Template-Sensor `battery_current_netto` = `battery_power_netto / battery_voltage_pack` (A, measurement), nur wenn V > 1.
   - Optional kleine Glättung/Hysterese (z.B. skip publish bei Δ<5 W), falls gewünscht.
3) **MQTT-Publish & HA-Discovery**
   - State-Topics (retain=false):
     - `b2500/1/battery/power_netto`
     - `b2500/1/battery/current_netto`
   - Discovery-Configs (retain=true) unter `homeassistant/sensor/<id>/config`, z.B.:
     - Power: unique_id `b2500_batt_power_netto`, stat_t `b2500/1/battery/power_netto`, dev_cla `power`, stat_cla `measurement`, unit `W`.
     - Current: unique_id `b2500_batt_current_netto`, stat_t `b2500/1/battery/current_netto`, dev_cla `current`, stat_cla `measurement`, unit `A`.
   - Device block: ids `["b2500_1"]`, name "B2500".

Hinweise
- Vorzeichen prüfen: `out_total_power` muss positiv bei Entladung; sonst im Lambda invertieren.
- Topics bewusst "netto" statt "net" wählen, um keine Verwechslung mit Netzleistung zu provozieren.
- Discovery nur einmal (retain), States zyklisch (kein retain).