# zigbee2mqtt-zbt2-migration

Step-by-step guide to migrate Zigbee2MQTT from Home Assistant Connect ZBT-1 to ZBT-2 without re-pairing devices, by cloning the coordinator IEEE address.

> **Disclaimer:** This worked on my setup (Home Assistant OS, Zigbee2MQTT 2.14, `ember` adapter). Make backups first and use it at your own risk.

## Why this works

Both sticks use the `ember` adapter type in Zigbee2MQTT. A switch between adapters of the same type doesn't need re-pairing, as long as the new stick has the old coordinator's IEEE address before Z2M starts on it.

Don't count on Z2M to copy the IEEE address itself: on some EFR32 sticks that write fails silently and devices stop responding. So you write it yourself, on a computer, before the switch.

## Requirements

- Home Assistant with the Zigbee2MQTT app, currently running on a ZBT-1 with `adapter: ember`
- A new Home Assistant Connect ZBT-2
- A computer with Python 3 (the commands below are for macOS; on Linux the port is `/dev/ttyACM*`)
- A USB extension cable for the ZBT-2

## Placeholders

| Placeholder | Meaning |
| --- | --- |
| `<ZBT1-SERIAL>` | Serial number of your ZBT-1, as shown in its port path |
| `<ZBT2-SERIAL>` | Serial number of your new ZBT-2, the part after `ZBT-2_` in its port path |
| `<COORDINATOR-IEEE>` | IEEE address of your current coordinator (ZBT-1), e.g. `0x00124b00…` |
| `XXXX` | The ZBT-2's USB port name on your computer |

## Phase 1: Prepare and back up

1. In the Z2M web UI, go to Settings → About and note the coordinator IEEE, firmware version and Zigbee channel.
2. Make a full Home Assistant backup.
3. Make a Z2M backup: Settings → Tools → "Request Z2M backup". This is a zip of `/config/zigbee2mqtt` (`database.db`, `configuration.yaml`, `coordinator_backup.json`). Download it.
4. Make sure all devices are online, and fix flaky ones before the switch.
5. Set the Z2M log level to `debug` for the migration.

## Phase 2: Prepare the new ZBT-2 on your computer

1. Plug the new ZBT-2 into your computer. If it isn't on Zigbee firmware yet, install it with the Home Assistant Toolbox in Chrome or Edge (toolbox.openhomefoundation.org → Install Firmware → Zigbee).
2. Install the flasher in a virtual environment:

```bash
python3 -m venv ~/usf
~/usf/bin/pip install universal-silabs-flasher
```

3. Find the stick:

```bash
ls /dev/cu.usbmodem*
```

4. Write the old coordinator's IEEE address, without `0x`:

```bash
~/usf/bin/universal-silabs-flasher --device /dev/cu.usbmodemXXXX --probe-methods "ezsp:460800" write-ieee --ieee <COORDINATOR-IEEE-without-0x>
```

5. Check it: run the same command again. It should report that the IEEE already matches and skip the write.
6. If the flasher asks for `--force`, stop and read up first: on some firmware that makes the write permanent.

## Phase 3: Switch over in Home Assistant

Allow 30–60 minutes. Keep this order so Z2M never starts against a missing stick.

1. Unplug the ZBT-2 from your computer.
2. Stop the Zigbee2MQTT app. A clean stop writes a fresh `coordinator_backup.json`. Don't delete or rename it: Z2M uses it to restore the network onto the new stick.
3. Unplug the ZBT-1.
4. Plug the new ZBT-2 into a USB 2 port on an extension cable, away from USB 3 devices and other radios.
5. Ignore the setup prompts Home Assistant shows for the discovered stick. Don't set up ZHA and don't reflash anything.
6. Find the new port under Settings → System → Hardware → All hardware (search for "ZBT-2").
7. Edit `/config/zigbee2mqtt/configuration.yaml` (e.g. with the File editor app) and replace the `serial:` block. Keep `network_key`, `pan_id`, `ext_pan_id` and `channel` as they are.
8. Start Zigbee2MQTT and read its log.

Old block (keep it for rollback):

```yaml
serial:
  baudrate: 115200
  rtscts: false
  port: /dev/serial/by-id/usb-Nabu_Casa_SkyConnect_v1.0_<ZBT1-SERIAL>-if00-port0
  adapter: ember
```

New block:

```yaml
serial:
  baudrate: 460800
  rtscts: true
  port: /dev/serial/by-id/usb-Nabu_Casa_ZBT-2_<ZBT2-SERIAL>-if00
  adapter: ember
```

Set the port only in this file and leave the serial section on the Zigbee2MQTT app's Configuration tab empty: values there override the file. Don't leave the port empty either. If you have more than one radio, autodetection can pick the wrong one.

## Phase 4: Verify (over 24 hours)

- [ ] The Z2M log shows the network restored from backup and the coordinator IEEE `<COORDINATOR-IEEE>`. A different IEEE → stop and roll back.
- [ ] Mains-powered devices (lights, plugs, routers) respond within minutes.
- [ ] Battery devices reconnect on their next wake-up. Press a button on remotes to speed this up.
- [ ] Unresponsive device: run Reconfigure in Z2M and press its pair button. Re-pair only as a last resort. If re-pairing fails, remove the device from Z2M's configuration with Z2M stopped first.
- [ ] The network map looks normal after a few hours.
- [ ] Automations that use Zigbee devices still fire.
- [ ] Z2M log level set back to `info`.

## Rollback

Roll back only within a few hours of the switch. After that, the ZBT-1's copy of the network falls too far behind.

1. Stop Z2M.
2. Paste the old `serial:` block back into `configuration.yaml`.
3. Unplug the ZBT-2, plug in the ZBT-1, start Z2M.

## Afterwards

- Don't plug the ZBT-1 back in as a coordinator: it still has the same IEEE address and network. Reflash it (e.g. to Thread) before reusing it.
- Remove the old ZBT-1 device in Home Assistant.

## Notes and pitfalls

- **Baud rate** is the USB serial speed between Home Assistant and the stick, not the Zigbee radio speed. It must match the firmware: ZBT-1 uses 115200, ZBT-2 uses 460800. A mismatch means Z2M can't start.
- **`rtscts: true`** turns on hardware flow control. On the ZBT-2 it prevents data loss, e.g. Z2M crashing during device firmware updates.
- **zsh comments:** a `# comment` pasted after a command makes zsh treat the words as filenames. Leave comments out.
- **Flasher not found:** install it in a venv (Phase 2) and call it as `~/usf/bin/universal-silabs-flasher`.
- **Flasher 1.3.0** replaced `--ezsp-baudrate 460800` with `--probe-methods "ezsp:460800"`.
- **IEEE format:** `--ieee` takes the address without `0x`, with or without colons.
- **Several ZBT-2s:** the serial number is the part after `ZBT-2_` in the port path. Settings → System → Hardware → All hardware lists them all.

## Sources

- [Zigbee2MQTT – EmberZNet adapters (ZBT-2 settings)](https://www.zigbee2mqtt.io/guide/adapters/emberznet.html)
- [Zigbee2MQTT FAQ – switching adapters](https://www.zigbee2mqtt.io/guide/faq/)
- [Nabu Casa – Is Zigbee2MQTT supported by ZBT-2?](https://support.nabucasa.com/hc/en-us/articles/31346954875421-Is-Zigbee2MQTT-supported-by-Home-Assistant-Connect-ZBT-2)
- [HA Community – ZBT-1 to ZBT-2 Z2M migration](https://community.home-assistant.io/t/zbt-1-to-zbt-2-z2m-migration-to-z2m/955320)
- [Z2M discussion #31908 – IEEE write failing silently](https://github.com/Koenkk/zigbee2mqtt/discussions/31908)
- [universal-silabs-flasher on PyPI](https://pypi.org/project/universal-silabs-flasher/)
