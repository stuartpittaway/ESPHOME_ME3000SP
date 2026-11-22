# ESPHOME_ME3000SP
Home Assistant (ESP Home) configuration for communication with SOFAR ME3000SP inverter over MODBUS/RS485

Configuration file uses ESP Home and an ESP8266 with RS485 chip for communication with SOFAR ME3000SP inverter (for battery storage).

The interface works in both directions:

* **Monitoring:** reads the inverter's live data (battery, grid, load, generation, daily and lifetime energy, temperatures, faults) into Home Assistant, ready for the energy dashboard.
* **Control:** force the battery to charge or discharge at a set power, hold it in standby, or return it to automatic self-consumption, from Home Assistant or its automations. See [Controlling charge and discharge](#controlling-charge-and-discharge).

Other devices supporting ESP Home and RS485 communication should work without significant change to the configuration.

![Screenshot of an example home automation dashboard](Dashboard-Example.png)

## Hardware

This configuration was built and tested on deliberately old, cheap hardware:

* **ESP8266 NodeMCU** development board
* **MAX3485 RS485 transceiver**, powered from the NodeMCU's **3.3 V** pin

**The RS485 module must run at 3.3 V.** The ESP8266's pins are 3.3 V only. Many RS485 modules are built around the 5 V MAX485 chip, and powering one from 5 V puts 5 V on the ESP8266's RX pin, which can damage it. Use a 3.3 V transceiver such as the MAX3485 (or a module that states it works at 3.3 V), and power it from the 3.3 V pin, not 5 V or VIN.

Wiring, matching the `uart` and `modbus` sections of `sofarme3000sp.yaml`:

| MAX3485 | NodeMCU | Notes |
|---|---|---|
| VCC | 3V3 | **Not** 5 V / VIN |
| GND | GND | |
| DI (driver input) | TX (GPIO1) | |
| RO (receiver output) | RX (GPIO3) | |
| DE and RE (joined) | D1 (GPIO5) | Flow control: switches the transceiver between sending and receiving |
| A / B | Inverter RS485 terminal | If nothing is received, try swapping A and B |

The RS485 link uses the ESP8266's main hardware serial port (GPIO1/GPIO3), which is also connected to the board's USB serial chip. That's why logging is moved to UART1 (`hardware_uart: UART1`, `baud_rate: 0`) in the config. Disconnect the transceiver if you ever need to flash the board over USB; after that, updates go over WiFi.

Other ESPHome boards with a 3.3 V RS485 transceiver should work with only the pin numbers changed.

## Controlling charge and discharge

The configuration can tell the inverter when to charge and discharge, using SOFAR's "passive mode" commands (the same commands as the [Sofar2mqtt](https://github.com/cmcgerty/Sofar2mqtt) project).

This is optional. All the sensors work whatever mode the inverter is in; you only need Passive Mode if you want Home Assistant to control charging and discharging.

### Switch the inverter to Passive Mode

The inverter only accepts remote charge and discharge commands in Passive Mode. Until you switch it over, every command is rejected and Battery Command Result shows `Invalid work mode (is the inverter in Passive Mode?)`.

On the inverter's own display:

1. Open **Settings** and select **12. Work Mode Set**.
2. Select **4. Set Passive Mode** and press **OK**.

In Passive Mode the inverter only does what it's told, and stops managing the battery on its own. Set **Battery Mode** to **Auto** in Home Assistant to get its normal self-consumption behaviour (charge from surplus solar, discharge to cover the house load) while still allowing remote control.

To stop using remote control, go back to **12. Work Mode Set** and select **1. Set Auto Mode**. The inverter then manages the battery itself again, and ignores the controls in Home Assistant.

### Controls in Home Assistant

Home Assistant gets these entities:

| Entity | What it does |
|---|---|
| Battery Mode | **Auto** (normal self-consumption), **Charge**, **Discharge** or **Standby** (neither charge nor discharge) |
| Charge Power | Charge rate in watts (100–3000 W) used when Battery Mode is Charge |
| Discharge Power | Discharge rate in watts (100–3000 W) used when Battery Mode is Discharge |
| Battery Command Result | The inverter's reply to the last command, for example `OK, charge enabled` or `Invalid work mode (is the inverter in Passive Mode?)` |

Changing a power level while that mode is active re-sends the command straight away. The selected mode and power levels are remembered, and re-sent about 15 seconds after the ESP restarts.

To charge overnight on a cheap tariff, for example, use a Home Assistant automation that sets Battery Mode to Charge at the start of the cheap period and back to Auto at the end.

### Example: charging on Intelligent Octopus

This automation uses the [Octopus Energy integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy). It charges the battery when either of these is true:

* the tariff is in its off-peak period, or
* Octopus has given you a cheap slot **and** the car is actually charging (drawing more than 1000 W).

At any other time, it puts the battery back in Auto.

```yaml
alias: Charge home battery during off-peak
description: >
  Force-charge the SOFAR battery during off-peak, or during an Intelligent
  Octopus slot while the car is actually charging
mode: restart
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.octopus_energy_electricity_<meter_serial>_<mpan>_off_peak
      - binary_sensor.octopus_intelligent_slot
    to: ["on", "off"]
  - trigger: numeric_state
    entity_id: sensor.power_evse
    above: 1000
  - trigger: numeric_state
    entity_id: sensor.power_evse
    below: 1000
    for: "00:02:00"
  - trigger: homeassistant
    event: start
actions:
  - choose:
      - conditions:
          - condition: or
            conditions:
              - condition: state
                entity_id: binary_sensor.octopus_energy_electricity_<meter_serial>_<mpan>_off_peak
                state: "on"
              - condition: and
                conditions:
                  - condition: state
                    entity_id: binary_sensor.octopus_intelligent_slot
                    state: "on"
                  - condition: numeric_state
                    entity_id: sensor.power_evse
                    above: 1000
        sequence:
          - action: number.set_value
            target:
              entity_id: number.sofarme3000sp_charge_power
            data:
              value: 3000
          - action: select.select_option
            target:
              entity_id: select.sofarme3000sp_battery_mode
            data:
              option: Charge
    default:
      - action: select.select_option
        target:
          entity_id: select.sofarme3000sp_battery_mode
        data:
          option: Auto
```

Before you use it, replace these entity IDs with your own (find them under Settings > Devices & services > Entities):

| Entity in the example | Replace with |
|---|---|
| `binary_sensor.octopus_energy_electricity_<meter_serial>_<mpan>_off_peak` | Your electricity meter's **Off Peak** binary sensor (search for "off peak") |
| `binary_sensor.octopus_intelligent_slot` | A binary sensor that is on during an Intelligent Octopus cheap slot |
| `sensor.power_evse` | Your car charger's power sensor, in watts. If it reports kW, change `1000` to `1` |
| `number.sofarme3000sp_charge_power`, `select.sofarme3000sp_battery_mode` | Only needed if you renamed the ESPHome device |

How it works:

* Any one trigger runs the automation: either Octopus sensor turning on or off, the car's power crossing 1000 W, or Home Assistant starting. The conditions then check the current state of all three entities and choose Charge or Auto, whichever trigger fired.
* The car-power triggers fire only when 1000 W is crossed, not on every reading, so the inverter isn't sent a command every few seconds. The "below" trigger waits 2 minutes, so a brief dip while the car is charging doesn't switch the battery back to Auto.
* If the car's power sensor is unavailable, the slot condition fails, so the battery doesn't charge on a slot alone.
* Charging at 3000 W stops by itself when the battery is full. Battery Command Result then shows `battery full`.
* The automation overrides any mode you set by hand the next time one of its triggers fires.

Octopus only bills a slot outside the normal off-peak hours at the cheap rate if the car actually charges during it. That's why the example checks the car's power, not just the slot sensor. Check your first few bills to confirm.

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

Several inverters can share the same RS485 bus, as long as each one has a different RS485 address. To add a second inverter:

1. **Set a unique address on the new inverter.** On the inverter, go to Settings > 5. Set Communication Addr. and choose an address no other inverter uses, for example `2`. The current address is shown under System Information > RS485 Address.
2. **Wire it into the bus.** Connect the inverters as a daisy chain (ESP RS485 module → inverter 1 → inverter 2), with a 120 Ω terminating resistor at each end of the chain. Keep any branches off the main line short.
3. **Enable it in the config.** In `sofarme3000sp.yaml`, uncomment the `inverter2` block and set `modbus_address` to the address from step 1:

   ```yaml
     inverter2: !include
       file: packages/sofar_me3000sp.yaml
       vars:
         name_prefix: "Inverter 2 "
         id_prefix: inverter2
         modbus_address: 2
   ```

4. **For a third inverter or more**, copy the block as `inverter3`, with its own `id_prefix` (`inverter3`), `name_prefix` (`"Inverter 3 "`) and address.
5. **Validate and install** in ESPHome Device Builder. In Home Assistant, check that the new "Inverter 2 …" entities appear and that the first inverter's entities are unchanged.

The first inverter uses an empty `name_prefix`, so its entities keep their original names and history. Don't add a prefix to it later, or Home Assistant will create new entities and the old ones will lose their history.

## Secrets

The configuration reads its credentials from ESPHome's `secrets.yaml`. Add these entries before validating:

```yaml
wifi_ssid: "your-wifi-name"
wifi_password: "your-wifi-password"
ap_fallback_password: "fallback-hotspot-password"   # at least 8 characters
sofarme3000sp__encryption_key: "base64-api-key"
```

The API encryption key is also used to encrypt OTA updates, so no separate OTA password is needed. Generate a key with `openssl rand -base64 32`.

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
