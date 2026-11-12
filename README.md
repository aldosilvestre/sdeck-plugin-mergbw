# sdeck-plugin-mergbw

A MeRGBW Hexagon Light plugin for [sdeck-plus](../sdeck-plus) (protocol v1, Python 3 standard
library only) that writes MeRGBW-style packets to a Bluetooth LED hexagon light.

> **The protocol is taken from the reference implementation.** Hand test on one Hexagon Light
> TG609 (2026-11-12): `power-off` turned the light off, which is the only effect that was observed.
> The other actions' packets were accepted by the light, but their visual effect was not reported,
> and the reconnect retry below was added after a rejection seen on that device and has not been
> re-tested live. The automated tests use a fake `busctl` only. Success messages say packets were
> *written*; the light's state is never confirmed (the reference never reads notifications, and
> neither does this plugin).

## Credit

The packet format comes from the documented protocol of
[jmbwell/mergbw](https://github.com/jmbwell/mergbw) (HEXAGON profile), MIT licensed, Copyright (c)
2025 John Burwell. No code or scene table was copied; the packet builder is written from the
documented facts.

## Actions

Every action has the required text setting `address` (`AA:BB:CC:DD:EE:FF`) and works on a key or a
dial press.

| Action | Extra settings | What is written |
|---|---|---|
| `power-on` / `power-off` | none | One power packet. |
| `set-color` | `hue` 0-359 (0), `saturation` 0-100 (100), `brightness` 0-100 (optional, no default) | Power on, then colour. A brightness packet follows only when you set `brightness`, so choosing a colour leaves the current brightness alone. |
| `set-brightness` | `brightness` 0-100 (100) | One brightness packet; power state untouched. |
| `set-scene` | `scene` 1-117 (2) | Power on, then the scene packet and a second packet with the reference's default value `0x3200`. The reference is uncertain what that second packet means, so it is not a setting. |

Percentages are sent as value x 10.

## Dependencies

The app shows these in the plugin's sidebar item and flags missing required commands. It never
installs anything for you.

| Kind | Name | Needed for | Notes |
|---|---|---|---|
| command | `python3` | everything | 3 (tested with 3.14); standard library only |
| command | `busctl` | everything | systemd's D-Bus client; talks to BlueZ |
| service | BlueZ (bluetoothd) | everything | system D-Bus |
| service | Bluetooth adapter | everything | powered, managed by BlueZ |
| service | Hexagon Light known to BlueZ | everything | paired or discovered with `bluetoothctl`; never scanned or paired by the plugin |

Also: the official MeRGBW app must be closed, because the light accepts one connection at a time.

## Limits

- The light drops its Bluetooth link when idle, so the first press after a pause reconnects and
  may take longer. Readiness is always decided from a fresh BlueZ read (connected **and**
  services resolved) before the first write, never from a snapshot taken before connecting.
- A write the light rejects with ATT error `0x0e` (seen right after a reconnect) is retried
  **once** per press, after another fresh readiness check, and only if at least 1 s of the 4.5 s
  budget is left; the success message then says "after 1 retry". Timeouts, other errors and
  connect failures are never retried.
- Connecting and resolving services can exceed the host's 5 s request limit on the **first press**;
  the plugin stops at 4.5 s, and a later press then works.
- The plugin does not disconnect afterwards, so the light stays connected to this computer and the
  official app cannot connect until you run `bluetoothctl disconnect <address>`.
- Only the GATT characteristic `0000fff3-...` is written, with BlueZ choosing the write type. The
  notify characteristic is never subscribed.
- No credentials are involved.

## Install

Copy `manifest.json` and `run` to `<data dir>/sdeck-plus/plugins/mergbw/` (usually
`~/.local/share/sdeck-plus/plugins/mergbw/`), keep `run` executable, rescan plugins and enable
**MeRGBW Hexagon Light**. Process separation is **not** sandboxing: the plugin runs with your
privileges, so read `run` before enabling it.

## Tests

```sh
python3 -B -I -m unittest discover -s tests -v
```

## Compatibility

- Plugin protocol: **1**. The `requirements` list in `manifest.json` needs an sdeck-plus build
  that knows that field (older builds reject unknown manifest fields). Without it, delete the
  `requirements` array: the plugin does not need it to run.
- Only `manifest.json` and `run` are needed at runtime; the `tests/` folder is for development.

## Validate

From an sdeck-plus checkout, validate this project without running it:

```sh
cargo run --offline --example plugin_validate -- /path/to/sdeck-plugin-mergbw
```

It prints `PLUGIN_VALIDATE_OK:` with the actions and a PATH check of the commands above. The
check only looks for the commands; the app never installs anything.
