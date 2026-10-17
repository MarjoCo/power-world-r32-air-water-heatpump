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

A notification is sent when a fault has stayed on for 6 seconds. Faults during the first minute after a Home Assistant restart are ignored, because readings are not reliable yet. That check uses `sensor.last_boot` from the System Monitor integration if you have it. Without it, the check simply passes.

### Linkage switch, EVU signal and SG signal need an external switch

`binary_sensor.linkage_switch`, `binary_sensor.evu_signal` and `binary_sensor.sg_signal` read the heat pump's external control inputs. They show something meaningful **only if you have wired an external switch or contact to those terminals** on the heat pump's control board:

- **Linkage switch:** an external on/off permission, for example from a room thermostat or a relay controlled by Home Assistant.
- **EVU signal:** the utility's blocking signal.
- **SG signal:** the Smart Grid signal.

Without anything connected, these inputs show a fixed default value that means nothing. Ignore them, or remove them from your dashboard. See the heat pump's installation manual for the terminals and how to wire them.

### Controls

The work mode, operation mode, setpoints and vacation mode in the dashboard are helpers (`input_select`, `input_number`, `input_boolean`) that follow the heat pump:

- **Heat pump → dashboard:** when a value changes on the heat pump's own panel, the dashboard follows. At Home Assistant start the dashboard waits until the values have been read.
- **Dashboard → heat pump:** when you change a value in the dashboard, it is written to the heat pump, but only if it differs from the current value. Automations never write, so syncing the dashboard can never cause a write loop.

The flag registers sometimes read as 0 for a moment. Mode changes are ignored while a register reads 0, so a read glitch cannot change the heat pump's mode.

### Automation names

The automations are named `Ultimate …` for compatibility with earlier installations of this package. The names have no effect on how they work.

## Upgrading from an earlier version

Entity IDs and unique IDs are unchanged, so history and dashboards keep working.

1. Remove the old package files: the folder with `automations.yaml`, `customize.yaml`, `input.yaml` and `ultimate.yaml`, plus the old Modbus file for the heat pump.
2. Install `packages/power_world_r32.yaml` as in steps 3–4.
3. Restart Home Assistant.

Changes in this version:

- **Fault notifications are sent once.** Er03, Er06 and anti-freeze used to trigger both the normal and the critical notification. The water-flow switch no longer sends a separate notification about 10 s before Er03.
- **The fault log text and its clear button are gone.** The log stopped recording after about five entries, because a text helper holds at most 255 characters. They are replaced by `binary_sensor.heat_pump_fault`, which disappears from the screen by itself. You can delete the leftover entities `input_text.fault_register` and `input_boolean.clear_fault_register` in Settings → Entities.
- **Modern Home Assistant syntax** (`triggers:` / `actions:`).

## Files

| File | Purpose |
|---|---|
| `packages/power_world_r32.yaml` | The whole integration: Modbus, sensors, helpers, automations and customize |
| `dashboards/power_world_r32_dashboard.yaml` | Dashboard: Overview (picture), Details, Error log, Defrosting, Energy |
| `www/power_world_r32/heat_pump.png` | Background picture for the dashboard |
