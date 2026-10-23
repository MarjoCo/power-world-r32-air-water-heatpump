# Power World R32 air-water heat pump for Home Assistant

A Home Assistant package for the Power World R32 air-water heat pump, read and controlled over Modbus. It includes:

- **Readings:** temperatures, compressor frequency and current, fan speeds, valve positions and defrost parameters.
- **Status:** the compressor, defrosting, the 3-way valve, the water pump and the electric heaters.
- **Faults:** every fault code (Er03, Er05, Er06 …) and inverter fault as its own sensor, plus one combined `binary_sensor.heat_pump_fault`.
- **Controls:** work mode, operation mode (Smart / Powerful / Silent), setpoints, temperature differences and vacation mode.
- **Notifications:** a push message when a fault appears. Water-flow, low-pressure and anti-freeze faults are sent as critical alerts.
- **Dashboard:** a picture of the heat pump with live values, plus details, fault log, defrosting and energy views.

Everything except the dashboard is in one file: `packages/power_world_r32.yaml`.

> **⚠️ Protect the heat pump's memory.** The heat pump saves every parameter you write to its flash memory, and the chip wears out after too many writes. This package writes to the heat pump **only when a person changes a value in the dashboard**, and only if the value differs from what the heat pump already has. Never add your own automations that write to the heat pump on a timer or after every sensor update.

## What you need

- Home Assistant 2024.10 or newer.
- An **RS485 → Ethernet/WiFi converter** with a Modbus TCP gateway (for example Elfin EW11, USR-TCP232 or Waveshare RS485 to ETH), wired to the heat pump's RS485 (A/B) terminals.
- [HACS](https://hacs.xyz/) with these frontend cards, which the dashboard needs: **card-mod**, **multiple-logbook-card** (Error log view) and **history-explorer-card** (Defrosting and Energy views).
- A notification service, for example the Home Assistant Companion app on your phone.

## Installation

### 0. Download the files

You do not need git. Open the [latest release](https://github.com/MarjoCo/power-world-r32-air-water-heatpump/releases/latest), download **power-world-r32-v….zip** under *Assets* and unpack it on your computer. You need three files from it:

- `packages/power_world_r32.yaml`
- `dashboards/power_world_r32_dashboard.yaml`
- `www/power_world_r32/heat_pump.png`

Copy them to Home Assistant's configuration folder (the one with `configuration.yaml`) with the **File editor** or **Samba share** add-on, or with **Studio Code Server**. The steps below say where each file goes.

### 1. Set up the RS485 converter

Configure the converter's serial side for the heat pump:

| Setting | Value |
|---|---|
| Baud rate | 9600 |
| Data bits / parity / stop bits | 8 / None / 1 (8N1) |
| Protocol | Modbus TCP ↔ Modbus RTU (gateway mode) |
| TCP port | 502 |

Give the converter a **fixed IP address**, either a DHCP reservation in your router or a static IP. Write the address down. The heat pump's Modbus slave address is `1`.

### 2. Turn on packages in Home Assistant

Open `configuration.yaml`. If it does not already load packages, add:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

If you already have a `homeassistant:` section, add only the `packages:` line under it. Do not create a second `homeassistant:` section.

### 3. Copy the package and set your IP address

1. Create the folder `packages` next to `configuration.yaml`, if it does not exist yet.
2. Copy `packages/power_world_r32.yaml` into it.
3. Open the file and find the lines marked `CHANGE`:

```yaml
modbus:
  - name: ultimate
    type: tcp
    host: 192.168.1.100   # <-- CHANGE: IP address of your RS485-to-TCP converter
    port: 502            # <-- CHANGE if your converter uses another port
```

Set `host` to your converter's IP address. Do not change `name: ultimate`, because the automations write to the heat pump through that name.

If your `configuration.yaml` already has a `modbus:` section for other devices, you can keep it. Home Assistant combines it with the one in the package.

### 4. Choose where notifications go

Fault messages are sent with `notify.notify`, which goes to the first notification service Home Assistant finds. To send them to a specific phone, replace `notify.notify` in the package (two places) with your phone's service, for example `notify.mobile_app_my_phone`. You can find the name in **Developer tools → Actions** by typing `notify.`.

To send to several phones, create a notify group in `configuration.yaml` and use its name:

```yaml
notify:
  - name: heat_pump
    platform: group
    services:
      - service: mobile_app_phone_1
      - service: mobile_app_phone_2
```

Then use `notify.heat_pump` in the package.

Critical alerts (water flow Er03, low pressure Er06 and secondary anti-freeze) use the iPhone critical-alert sound, which plays even when the phone is muted. On Android they arrive as normal notifications.

### 5. Add the dashboard

1. Copy `www/power_world_r32/heat_pump.png` to `www/power_world_r32/` next to `configuration.yaml`. Create the folders if they do not exist. If `www` did not exist before, restart Home Assistant once so it starts serving the folder.
2. Install **card-mod**, **multiple-logbook-card** and **history-explorer-card** from HACS (Frontend).
3. Go to **Settings → Dashboards → Add dashboard → New dashboard from scratch**, name it (for example "Heat pump") and open it.
4. Top right **⋮ → Edit dashboard → ⋮ → Raw configuration editor**. Replace everything with the contents of `dashboards/power_world_r32_dashboard.yaml` and save.

### 6. Check the configuration and restart

1. **Developer tools → YAML → Check configuration.** It must say the configuration is valid.
2. **Settings → System → Restart Home Assistant.** A restart is needed the first time, because Modbus cannot be reloaded on its own.

### 7. Check that it works

After the restart, wait a minute and check in **Developer tools → States**:

- `sensor.water_inlet_temperature` and `sensor.ambient_temperature` show real temperatures.
- `sensor.work_mode` shows a number from 0 to 4.
- `binary_sensor.heat_pump_fault` is `off`, unless the heat pump really has a fault.

If the values are `unavailable`:

- Check the IP address and port, and that Home Assistant can reach the converter.
- Check the converter's serial settings (9600 8N1) and gateway mode.
- If nothing comes through at all, swap the A and B wires.
- Look for Modbus errors in **Settings → System → Logs**.

## How it works

### Faults

Each fault code has its own `binary_sensor` with `device_class: problem`. `binary_sensor.heat_pump_fault` is `on` while any fault is active. Its attribute `latest` holds the name of the fault that came on most recently, with `(+N)` added when several are active. The dashboard shows that text only while a fault is active, and it disappears by itself when the fault clears.

A notification is sent when a fault has stayed on for 6 seconds. A fault that is already active when Home Assistant starts is reported once, after the first reading. The package no longer reads `sensor.last_boot`; System Monitor is not needed.

### Linkage switch, EVU signal and SG signal need external wiring

`binary_sensor.linkage_switch`, `binary_sensor.evu_signal` and `binary_sensor.sg_signal` show the state of input terminals on the heat pump's main board. They mean something **only if you have wired a contact to those terminals**. Nothing needs to be set in Home Assistant.

**Linkage switch** (input IN4 on the main board; the controller manual calls it "Switch on-off function required for AC"):

- When the contact is closed (short-connected), heating and cooling are allowed. The heat pump's own controller still decides when the compressor runs.
- When the contact is open, the heat pump does not heat or cool.
- You can wire a relay here, for example a Shelly, to give or remove heating permission from Home Assistant.
- The user manual says the heat pump must not be switched on and off more than 4 times within one hour.

**EVU signal and SG signal** are used only by the heat pump's Smart Grid function. It is off by default (parameter S01 = 0). When S01 = 1, the two signals select the operating state, according to the user manual:

| SG | EVU | Operating state |
|---|---|---|
| ON | ON | Increased operation: hot water has priority, setpoint 70 °C, tank heater on below 69 °C |
| OFF | ON | Increased operation: hot water has priority with the normal setpoint |
| ON | OFF | Normal operation |
| OFF | OFF | Decreased operation: no hot water, tank heater and disinfection off; heating/cooling runs at most "SG operating time" (S2), then the unit stops |

The manual adds: *"To use this function, the auxiliary electric heater for DHW tank must be installed. Otherwise, leave the EVU interface disconnected."*

If nothing is wired to these terminals, or S01 = 0, the three sensors show a fixed value that means nothing. Ignore them or remove them from your dashboard.

### Controls

The work mode, operation mode, setpoints and vacation mode in the dashboard are helpers (`input_select`, `input_number`, `input_boolean`) that follow the heat pump:

- **Heat pump → dashboard:** when a value changes on the heat pump's own panel, the dashboard follows. At Home Assistant start the dashboard waits until the values have been read.
- **Dashboard → heat pump:** when you change a value in the dashboard, it is written to the heat pump, but only if it differs from the current value. Automations never write, so syncing the dashboard can never cause a write loop.

The flag registers sometimes read as 0 for a moment. Mode changes are ignored while a register reads 0, so a read glitch cannot change the heat pump's mode.

### Setpoint ranges

The allowed setpoint ranges differ between Power World documents, apparently by controller version:

| Parameter | Newer user manual | Communication protocol, controller manual, 2023 user manual |
|---|---|---|
| P05 Heating | 15–55 °C | 15–50 °C |
| P03 Hot water | 28–55 °C | 28–60 °C |
| P04 Cooling | 7–30 °C | 7–30 °C |

The sliders follow the newer user manual. Check the parameter table in **your own unit's manual**, and if it differs, change `min:` / `max:` of `input_number.heating_set_temperature` and `input_number.hot_water_set_temperature` in the package to match. If the heat pump has a value outside the slider's range, the dashboard cannot show it, and Home Assistant logs an error.

### Known limitation: lost connection shows as "off"

If Home Assistant loses the Modbus connection to the heat pump, the fault sensors and `binary_sensor.compressor` show `off` instead of `unavailable`. During a connection break, a fault would therefore not show or notify. The temperature sensors do show `unavailable`. If you want an alert for a lost connection, make an automation that triggers when `sensor.water_inlet_temperature` has been `unavailable` for a few minutes.

### Automation names

The automations are named `Power World …`. Earlier versions of this package called them `Ultimate …`. For compatibility with those installations, the automation IDs (`ultimate_…`) and the Modbus hub name (`ultimate`) are unchanged. If you upgrade, your existing automation entity IDs stay the same, and only the names shown in the UI change.

## Upgrading from an earlier version

Entity IDs and unique IDs are unchanged, so history and dashboards keep working.

1. Remove the old package files: the folder with `automations.yaml`, `customize.yaml`, `input.yaml` and `ultimate.yaml`, plus the old Modbus file for the heat pump.
2. Install `packages/power_world_r32.yaml` as in steps 3–4.
3. Restart Home Assistant.

Changes in this version:

- **Fault notifications are sent once.** Er03, Er06 and anti-freeze used to trigger both the normal and the critical notification. The water-flow switch no longer sends a separate notification about 10 s before Er03.
- **The fault log text and its clear button are gone.** The log stopped recording after about five entries, because a text helper holds at most 255 characters. They are replaced by `binary_sensor.heat_pump_fault`, which disappears from the screen by itself. You can delete the leftover entities `input_text.fault_register` and `input_boolean.clear_fault_register` in Settings → Entities.
- **The on/off switch no longer overwrites other settings.** `switch.u_energy_on_off` used to write the whole register 0x0000 as 65 (on) or 64 (off). That register also holds the A41, manual frequency, C01, B01 and A45 settings, so on a heat pump where those differ, the old switch changed them too. Now it changes only the on/off bit and writes only when the value really changes. The entity ID is unchanged. If you have used the old switch, check those settings on the heat pump's panel.
- **Cooling slider from 7 °C** (was from 12), as in every Power World manual. Heating (15–55 °C) and hot water (28–55 °C) are unchanged. See "Setpoint ranges" below.
- **Modern Home Assistant syntax** (`triggers:` / `actions:`).

## Support and maintenance

This package is shared as is, because it works on my own heat pump. **It is not actively maintained, and I cannot promise to answer questions or fix problems.** If something needs changing, you are welcome to fork the repository and change it yourself. The MIT license allows that.

You use the package at your own risk. Check that the settings suit your installation before you change anything on the heat pump.

## License

[MIT](LICENSE)

## Files

| File | Purpose |
|---|---|
| `packages/power_world_r32.yaml` | The whole integration: Modbus, sensors, helpers, automations and customize |
| `dashboards/power_world_r32_dashboard.yaml` | Dashboard: Overview (picture), Details, Error log, Defrosting, Energy |
| `www/power_world_r32/heat_pump.png` | Background picture for the dashboard |
