# Zigbee2MQTT Coordinator Auto-Recovery

A Home Assistant automation that brings Zigbee2MQTT (Z2M) back after a network coordinator outage, so you don't have to start the add-on by hand.

**You need this if** you run the Z2M add-on on Home Assistant OS with a network coordinator (`serial.port: tcp://...`, such as an SMLIGHT SLZB device), and Z2M stays stopped after the coordinator drops out and returns.

## The problem

Unplug a network coordinator for a couple of minutes, plug it back in, and Zigbee stays down. The coordinator is healthy and its port is open, but the add-on sits in state `error` until someone starts it. There is no notification, so the first sign is usually that your lights stopped working.

Anything that takes the coordinator away for more than about a minute can cause this: a knocked cable, a switch reboot, or a PoE or power interruption. Shorter interruptions usually recover by themselves.

## Why it happens

1. **Z2M exits when it loses the coordinator.** It logs `Adapter disconnected, stopping`. On restart it fails with `connect EHOSTUNREACH` and `Failed to start EZSP layer with status=HOST_FATAL_ERROR`, and each failed start takes only about 3.5 seconds on an Ember adapter.
2. **The Supervisor watchdog is rate limited** to 10 restarts per 30 minutes (`WATCHDOG_THROTTLE_MAX_CALLS` and `WATCHDOG_THROTTLE_PERIOD` in [`supervisor/apps/const.py`](https://github.com/home-assistant/supervisor/blob/main/supervisor/apps/const.py)). Failures that fast use all 10 in under a minute.
3. **Nothing retries after that.** The watchdog reacts to the add-on exiting. Once the limit is hit, that exit is ignored and no further exit ever comes, so the add-on stays stopped even after the 30 minutes pass.

The setting isn't configurable, and turning the watchdog off wouldn't help.

### What about Z2M's own watchdog?

Z2M has a [built-in watchdog](https://www.zigbee2mqtt.io/guide/installation/15_watchdog.html), set with the add-on's `watchdog` option, which retries without exiting. It is a reasonable alternative, with limits. The notes below come from its documentation and source, not from testing:

- Its list of delays is finite, so a long enough outage still ends in an exit.
- The default delays grow to an hour, so recovery can lag well behind the coordinator's return.
- It only runs after a successful first start, so it doesn't help if the coordinator is missing when Z2M starts.

## What the automation does

It starts the add-on when both of these are true:

- the Z2M bridge has been offline for 2 minutes
- the coordinator is reachable

It checks the moment the coordinator comes back, and every 5 minutes. The 5-minute check also covers Z2M being down after a reboot.

The action is `hassio.app_start`, which does nothing if the add-on is already running, so a false trigger can't interrupt Zigbee.

## Install

1. Make sure packages are enabled in `configuration.yaml`:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. Copy [`zigbee2mqtt_auto_recovery.yaml`](zigbee2mqtt_auto_recovery.yaml) to `/config/packages/`.
3. Change the entities to match your system (next section).
4. Check the configuration, then reload it (**Developer Tools > YAML > All YAML configuration**).

You can also paste the part under `automation:` into the automation editor's YAML mode.

## Adapt it to your system

| Item | Value in the file | What to do |
|---|---|---|
| Coordinator reachable sensor | `binary_sensor.slzb_mr3u_ethernet` | **Must be changed** in both places. See below |
| Z2M bridge sensor | `binary_sensor.zigbee2mqtt_bridge_connection_state` | Check it exists. This is the default name |
| Add-on slug | `45df7312_zigbee2mqtt` | The edge add-on has a different slug. Check the add-on's URL |

For the coordinator sensor:

- **SMLIGHT devices:** the [SMLIGHT integration](https://www.home-assistant.io/integrations/smlight/) provides an Ethernet sensor. This is the tested setup.
- **Anything else:** add the [Ping integration](https://www.home-assistant.io/integrations/ping/) for the coordinator's IP and use its binary sensor. This should work the same way, but I haven't tested it.

A sensor that goes `unavailable` while the coordinator is gone is fine, because the automation only looks for `on`.

**Older Home Assistant versions** may not have `hassio.app_start`. Check **Developer Tools > Actions**. If it's missing, use `hassio.addon_start` with `addon:` in place of `app:`. I haven't tested that form.

## Testing

Tested on 2026-11-06 with Z2M 2.14.3-1, Home Assistant Core 2026.10.2, HAOS 18.3, Supervisor 2026.10.5 and an SLZB-MR3U on PoE, by unplugging the coordinator for about 3.5 minutes:

| Time | Event |
|---|---|
| 15:12:58 | Z2M exits: `read ETIMEDOUT`, `Adapter disconnected, stopping` |
| 15:12:59 to 15:13:38 | Exactly 10 watchdog restarts |
| 15:13:41 | Final exit. No more restarts; add-on in `error` |
| ~15:16:13 | Coordinator back |
| 15:21:28 | Over 5 minutes later, add-on still in `error` (automation off for the test) |
| 15:21:33 | Automation turned on |
| 15:25:00 | Automation fires on its 5-minute check |
| 15:25:04 | `Zigbee2MQTT started!`, all 24 devices joined |

**Not yet tested:** the trigger that fires the moment the coordinator returns, because the automation was off at that point. If it didn't work, the 5-minute check would still catch it.

To test yours, unplug the coordinator for 3 minutes, plug it back in, and check that Z2M returns within about 5 minutes. Zigbee is down throughout.

Logs and the full timeline are in the [original bug report](https://github.com/danielrosehill/Bug-Reports-And-Fixes/tree/main/zigbee2mqtt/2026-11-06-tcp-coordinator-outage-no-self-recovery).

## Things to know

- **Stopping Z2M on purpose:** turn this automation off first, or it will start Z2M again within 5 minutes. This matters for coordinator firmware flashes.
- **Other causes of failure:** if Z2M won't start for another reason, such as a bad configuration, the automation retries every 5 minutes without fixing anything. That's harmless, but it won't tell you. Add a notification if you want one.
- **Broker outages:** the automation won't help if the Z2M bridge sensor is `unavailable`, for instance when Home Assistant has lost MQTT.

## Upstream status

This is a workaround. The proper fix belongs upstream:

- [Koenkk/zigbee2mqtt#33622](https://github.com/Koenkk/zigbee2mqtt/issues/33622): feature request to keep retrying network coordinators
- [home-assistant/supervisor#7190](https://github.com/home-assistant/supervisor/issues/7190): the watchdog leaves apps stopped without telling anyone

Earlier reports: [#21921](https://github.com/Koenkk/zigbee2mqtt/issues/21921), [#29654](https://github.com/Koenkk/zigbee2mqtt/issues/29654), and [hassio-zigbee2mqtt#760](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt/issues/760). A comment on #29654 suggested a similar automation based on the add-on's running sensor. This version adds the coordinator check and the periodic retry.

## License

[MIT](LICENSE)
