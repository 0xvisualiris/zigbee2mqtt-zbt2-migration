# zigbee2mqtt-zbt2-migration
Step-by-step guide to migrate Zigbee2MQTT from Home Assistant Connect ZBT-1 to ZBT-2 without re-pairing devices, by cloning the coordinator IEEE address.


# Zigbee2MQTT migration: ZBT-1 → ZBT-2

Nov 17, 2026

## Overview

Zigbee2MQTT moves from the ZBT-1 (SkyConnect) to a new ZBT-2 without re-pairing devices. Both sticks use the `ember` adapter type, so this works as long as the new stick gets the old coordinator's IEEE address before Z2M starts on it.

Z2M can't be trusted to copy the IEEE address itself: on some EFR32 sticks that write fails silently and devices stop answering. So it is written by hand on the Mac first.

| Stick | Serial | Role |
| --- | --- | --- |
| ZBT-1 (SkyConnect) | `<ZBT1-SERIAL>` | Old Zigbee coordinator, retired |
| ZBT-2 (new) | `<ZBT2-ZIGBEE-SERIAL>` | New Zigbee coordinator for Z2M |
| ZBT-2 (existing) | `<ZBT2-THREAD-SERIAL>` | Thread (OpenThread Border Router), don't touch |

Coordinator IEEE (from the ZBT-1): `<COORDINATOR-IEEE>`. New stick's factory IEEE: `<FACTORY-IEEE>`.

## Phase 1: Prepare and back up

1. In the Z2M web UI, go to Settings → About and note the coordinator IEEE, firmware version and Zigbee channel.
2. Make a full Home Assistant backup.
3. Make a Z2M backup: Settings → Tools → "Request Z2M backup". This is a zip of `/config/zigbee2mqtt` (`database.db`, `configuration.yaml`, `coordinator_backup.json`). Download it to the Mac.
4. Make sure all devices are online, and fix flaky ones before the switch.
5. Leave the Z2M log level on `debug` during the migration.

## Phase 2: Prepare the new ZBT-2 on the Mac

Done on 2026-11-10: the stick runs EmberZNet 7.5.1.0 and has the IEEE `<COORDINATOR-IEEE>`.

1. Plug the new ZBT-2 into the Mac. If it isn't on Zigbee firmware yet, install it with the Home Assistant Toolbox in Chrome or Edge (toolbox.openhomefoundation.org → Install Firmware → Zigbee).
2. Install the flasher in a virtual environment:

```bash
python3 -m venv ~/usf
~/usf/bin/pip install universal-silabs-flasher
```

3. Find the stick: `ls /dev/cu.usbmodem*` (here: `/dev/cu.usbmodemXXXX`).
4. Write the old coordinator's IEEE, without `0x`:

```bash
~/usf/bin/universal-silabs-flasher --device /dev/cu.usbmodemXXXX --probe-methods "ezsp:460800" write-ieee --ieee <COORDINATOR-IEEE-without-0x>
```

5. Check it: run the same command again. It should report that the IEEE already matches and skip the write.
6. If the flasher ever asks for `--force`, stop first: on some firmware that makes the write permanent.

## Phase 3: Switch over in Home Assistant

Allow 30–60 minutes. Keep this order so Z2M never starts against a missing stick.

1. Unplug the ZBT-2 from the Mac.
2. Stop the Zigbee2MQTT app. A clean stop writes a fresh `coordinator_backup.json`. Don't delete or rename it.
3. Unplug the ZBT-1.
4. Plug the new ZBT-2 into a USB 2 port on an extension cable, at least about 1 m from the Thread ZBT-2.
5. Ignore the "Konfigurieren" cards HA shows for the sticks. Don't set up ZHA and don't reflash anything.
6. Edit `/config/zigbee2mqtt/configuration.yaml` with the File editor and replace the `serial:` block. Keep `network_key`, `pan_id`, `ext_pan_id` and `channel` as they are.
7. Start Zigbee2MQTT and read its log.

Old block (keep for rollback):

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
  port: /dev/serial/by-id/usb-Nabu_Casa_ZBT-2_<ZBT2-ZIGBEE-SERIAL>-if00
  adapter: ember
```

Set the port only in this file. Leave the serial section on the Zigbee2MQTT app's Configuration tab empty, because values there override the file. Never leave the port empty: autodetection could grab the Thread stick.

## Phase 4: Verify (over 24 hours)

- [ ] Z2M log shows the network restored from backup and coordinator IEEE `<COORDINATOR-IEEE>`. A different IEEE → stop and roll back.
- [ ] Mains-powered devices (lights, plugs, routers) respond within minutes.
- [ ] Battery devices reconnect on their next wake-up; press a button on remotes and the Hue wall switches.
- [ ] Unresponsive device: run Reconfigure in Z2M and press its pair button. Re-pair only as a last resort; if re-pairing fails, remove it from Z2M's configuration with Z2M stopped first.
- [ ] Network map looks sane after a few hours.
- [ ] Automations still fire (motion lights, "Hue Wandschalter" blueprint).
- [ ] Z2M log level set back to `info`.

## Rollback and afterwards

Roll back only within a few hours of the switch. After that, the ZBT-1's copy of the network falls too far behind.

1. Stop Z2M.
2. Paste the old `serial:` block back into `configuration.yaml`.
3. Unplug the ZBT-2, plug in the ZBT-1, start Z2M.

Afterwards:

- Don't plug the ZBT-1 back in as a coordinator: it has the same IEEE and network. Reflash it (e.g. to Thread) before reusing it.
- Remove the old ZBT-1 device in HA.
- Give the new ZBT-2 the label `netzwerk_infrastruktur` and its area.

## Notes and pitfalls

- **Baud rate** is the USB serial speed between HA and the stick, not Zigbee radio speed. It must match the firmware: ZBT-1 uses 115200, ZBT-2 uses 460800. A mismatch means Z2M can't start.
- **rtscts: true** turns on hardware flow control. On the ZBT-2 it prevents data loss, e.g. Z2M crashes during device firmware updates.
- **zsh comments:** copying `# ...` after a command makes zsh treat the words as filenames. Leave comments out.
- **Flasher not found:** install it in a venv (Phase 2) and call it as `~/usf/bin/universal-silabs-flasher`.
- **Flasher 1.3.0** replaced `--ezsp-baudrate 460800` with `--probe-methods "ezsp:460800"`.
- **IEEE format:** `--ieee` takes the address without `0x`, with or without colons.
- **Which ZBT-2 is which:** the serial is the part after `ZBT-2_` in the port path. Settings → System → Hardware → All hardware lists both.

## Sources

- [Zigbee2MQTT – EmberZNet adapters (ZBT-2 settings)](https://www.zigbee2mqtt.io/guide/adapters/emberznet.html)
- [Zigbee2MQTT FAQ – switching adapters](https://www.zigbee2mqtt.io/guide/faq/)
- [Nabu Casa – Is Zigbee2MQTT supported by ZBT-2?](https://support.nabucasa.com/hc/en-us/articles/31346954875421-Is-Zigbee2MQTT-supported-by-Home-Assistant-Connect-ZBT-2)
- [HA Community – ZBT-1 to ZBT-2 Z2M migration](https://community.home-assistant.io/t/zbt-1-to-zbt-2-z2m-migration-to-z2m/955320)
- [Z2M discussion #31908 – IEEE write failing silently](https://github.com/Koenkk/zigbee2mqtt/discussions/31908)
- [universal-silabs-flasher on PyPI](https://pypi.org/project/universal-silabs-flasher/)
