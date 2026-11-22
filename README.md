# ESPHOME_ME3000SP
Home Assistant (ESP Home) configuration for communication with SOFAR ME3000SP inverter over MODBUS/RS485

Configuration file uses ESP Home and an ESP8266 with RS485 chip for communication with SOFAR ME3000SP inverter (for battery storage).

Other devices supporting ESP Home and RS485 communication should work without significant change to the configuration.

![Screenshot of an example home automation dashboard](Dashboard-Example.png)

## Files

* `sofarme3000sp.yaml`: the device config (board, WiFi, API, UART and Modbus bus)
* `packages/sofar_me3000sp.yaml`: the sensors for one inverter, included once per inverter

In Home Assistant, copy both into the ESPHome config folder, keeping the `packages` subfolder:

```
/homeassistant/esphome/sofarme3000sp.yaml
/homeassistant/esphome/packages/sofar_me3000sp.yaml
```

That is the path seen from the File editor, Studio Code Server, SSH or Samba add-ons; ESPHome Device Builder sees the same folder as `/config/esphome/`. Device Builder only lists files directly in that folder, so the package in `packages/` won't appear as a device, and you edit it with one of those add-ons instead.

## Multiple inverters

Several inverters can share the same RS485 bus, as long as each one has a different RS485 address. Set the address on each inverter (Settings > 5. Set Communication Addr.; the current address is shown under System Information), then uncomment the `inverter2` block in `sofarme3000sp.yaml` and set `modbus_address` to match.

The first inverter uses an empty `name_prefix`, so its entities keep their original names and history. Additional inverters get a prefix such as `"Inverter 2 "`.

Wire the bus as a daisy chain with a 120 Ω terminating resistor at each end.

## Secrets

The configuration reads its credentials from ESPHome's `secrets.yaml`. Add these entries before validating:

```yaml
wifi_ssid: "your-wifi-name"
wifi_password: "your-wifi-password"
ap_fallback_password: "fallback-hotspot-password"   # at least 8 characters
sofarme3000sp__encryption_key: "base64-api-key"
sofarme3000sp__ota_password: "ota-password"
```

## Sensors Reported into Home Assistant

The configuration has the correct units, icons and device classes to make integration with energy dashboards easy.

* Running State
* Running State Description
* Fault Message 1
* Fault Message 2
* Fault Message 3
* Fault Message 4
* Fault Message 5
* Grid Voltage
* Grid Current
* Grid Frequency
* Battery Power
* Battery Voltage
* Battery Current
* State of Charge
* Battery Temperature
* Feed Power
* Load Power
* Input Output Power
* Generation Power
* EPS Voltage
* EPS Power
* Energy Generation (Day)
* Energy Export (Day)
* Energy Import (Day)
* Energy Consumption (Day)
* Total generation
* Total export
* Total import
* Total load consumption
* Energy Battery Charge (Day)
* Energy Battery Discharged (Day)
* Count Down Timer
* Inverter Alert Message
* Battery Cycles
* Inverter Temperature
* Inverter Heat Sink Temperature
* Inverter Alert Description
