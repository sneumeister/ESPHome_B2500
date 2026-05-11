# ESPHome_B2500

Used original YAML from [esphome-b2500](https://tomquist.github.io/esphome-b2500/ "https://tomquist.github.io/esphome-b2500/").

Adaption to limit output of B2500 in accordance [OpenDTU-OnBattery](https://github.com/hoylabs/OpenDTU-OnBattery "https://github.com/hoylabs/OpenDTU-OnBattery")'s "limit_absolut" MQTT topic.

## Background

I originally had two solar panels connected to a Hoymiles HM-800.

Everything was controlled by [OpenDTU-OnBattery](https://github.com/hoylabs/OpenDTU-OnBattery "https://github.com/hoylabs/OpenDTU-OnBattery").... no limitation feeding into the public grid with the old Ferrari's electricity meter (ran backwards when more power was produced).

Since this was replaced by a digital two-way electricity meter, feeding excess power does not give any more benefit, so I bought a small energy storage Marstek B2500.

Now my configuration is:

- 2 solar panels connected to PV1 and PV2 input of the B2500.
- 2 DC1 / DC2 outputs of the B2500 connected to PV1 / PV2 input of the Hoymiles.

The recent power usage is read from a digital meter via [Tasmota-SML-Script](https://github.com/ottelo9/tasmota-sml-script "https://github.com/ottelo9/tasmota-sml-script") and published via MQTT.

[OpenDTU-OnBattery](https://github.com/hoylabs/OpenDTU-OnBattery "https://github.com/hoylabs/OpenDTU-OnBattery") reads this value and limits its AC-output to not feed power into the public grid. Excess power should be stored in the B2500 battery and used when more power is required or to cover my base load after sunset. <br>

This works in principle with an "open" B2500 (set to 800 W max. output), but it looks like I get regulation oscillations, when Hoymiles limits the DC-in to not feed too much power and on the other hand the voltage controller of the B2500 tries to achieve 800 W… however I got upward and downward deviations.

## Solution

I already had an ESP32 with online compiled [esphome-b2500](https://tomquist.github.io/esphome-b2500/ "https://tomquist.github.io/esphome-b2500/") running and visualization in HomeAssistant.

I wanted [OpenDTU-OnBattery](https://github.com/hoylabs/OpenDTU-OnBattery "https://github.com/hoylabs/OpenDTU-OnBattery") to be the master deciding about power to feed.

I did not understand `zeropower`, so I took `config.yaml`
and modified it (supported by [cursor.ai](https://www.cursor.ai "https://www.cursor.ai")) to read OpenDTU's MQTT-topic `solar/<ID>/status/limit_absolute` and use this value to limit the B2500 output (`timer 1`) to the same value. This avoids unnecessary regulation cycles and stabilizes the Hoymiles output.

## How it works

`config.yaml` line 105 defines the actual OpenDTU_OnBattery topic containing the power_limit, e.g. `solar/114184878827/status/limit_absolute` (adjust to your situation). <br>

This limit is then used as the limit for `timer 1` power. <br>
(`Timer 1` must be set to `00:00 - 23:59` and `active`.)

In consequence, the Hoymiles limit and the B2500 limit move in parallel.

## Additions for OpenDTU-OnBattery

Since [OpenDTU-OnBattery](https://github.com/hoylabs/OpenDTU-OnBattery "https://github.com/hoylabs/OpenDTU-OnBattery") offers the possibility to show certain energy storage data,

I added three additional topics:

* `b2500/sensor/b2500_-_1_-_battery_voltage_pack`  
  The voltage as sum of all single cell voltages; can also be found in the JSON of `b2500/1/battery/cell_voltage`.
* `b2500/sensor/b2500_-_1_-_battery_power_netto` / `b2500/1/battery/power_netto`  
  The current power in/out of the battery (- = charge, + = discharge). Value is calculated from `p_netto = b2500_total_power_out_1 - b2500_total_power_in_1`.
* `b2500/sensor/b2500_-_1_-_battery_current_netto` / `b2500/1/battery/current_netto`  
  The current that flows into (-) or out of (+) the battery; value is calculated from `p_netto / battery_voltage_pack`.

For OpenDTU that means:

`Einstellungen > Batterie`

- activate interface with `Batteriewerte aus MQTT Broker`
- Topic für SoC: `b2500/1/battery/remaining_percent`
- Topic für Spannung: `b2500/sensor/b2500_-_1_-_battery_voltage_pack`
- Topic für Strom: `b2500/1/battery/current_netto`

## How to compile new firmware

Precondition: recent [Python](https://www.python.org/ "https://www.python.org/") is installed.
Install [ESPHome](https://esphome.io/ "https://esphome.io/"):

```
PS> pip install esphome
```

Adjust the `example_secrets.yaml` and save as `secrets.yaml`.

Change the `esp32` section to your actual board and memory:

```yaml
esp32:
  board: "esp32dev"
  variant: "esp32"
  flash_size: 4MB
```

Compile firmware with the following command:

```
PS> python -m esphome compile config.yaml
```

Upload with:

```
PS> python -m esphome upload config.yaml
```

(you will be asked for OTA or COM-Port)

## References

- OpenDTU_OnBattery: <https://github.com/hoylabs/OpenDTU-OnBattery>
- esphome-b2500 by tomquist: <https://tomquist.github.io/esphome-b2500/>
- esphome-b2500 by tomquist: <https://github.com/tomquist/esphome-b2500>
- hm2500pub by noone2k: <https://github.com/noone2k/hm2500pub>
- ESPHome: <https://esphome.io/>
- tasmota-sml-script by otello9: <https://github.com/ottelo9/tasmota-sml-script>

## Remarks

Done with support of AI: [Cursor.com](https://www.cursor.com)
