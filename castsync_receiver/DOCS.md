# CastSync Receiver

Turns your Home Assistant box into an AirPlay speaker and a Spotify Connect
device, and feeds both into a [Snapcast](https://github.com/badaix/snapcast)
server so they play in tight sync across every room running a `snapclient`.

## Status: unbuilt

I wrote this add-on's Dockerfile, s6 service, and config against the current
upstream docs for shairport-sync, librespot and snapcast, but **I have not
been able to build or run it** — no Docker was available in the environment
that produced it. Treat the first install as a build attempt, not a known-good
release. If the Supervisor build log shows an error, that's expected to
happen at least once; the likely failure points are:

- An `apk add` package name that's moved or been renamed in the Alpine base
  version pinned in `build.yaml`.
- A shairport-sync/librespot/snapcast git tag that's since been superseded
  (the Dockerfile pins specific tags -- bump them if they 404).
- A `cmake`/`cargo`/`configure` flag name drifting between versions.

Share the build log and these are normally quick fixes.

## What's in, what isn't

| Source | Status |
|---|---|
| AirPlay (iPhone, Mac) | Built — shairport-sync, classic AirPlay 1, self-contained mDNS |
| Spotify Connect | Built — librespot, zeroconf pairing, no credentials stored in config |
| DLNA "Cast to device" (Android, Windows, VLC) | **Not in this build.** Needs a DLNA MediaRenderer (gmediarender) piping raw PCM into a snapserver `pipe://` source; the GStreamer sink flags needed weren't something I could confirm without a build environment, so I left it out rather than ship a guess. Config sections in the run script are set up to make adding it later a small diff — ask if you want this filled in. |
| YouTube / Netflix / other Cast senders | Not attempted — see the CastSync integration README for why (device attestation, DRM). |

## Why AirPlay 1, not AirPlay 2

AirPlay 2 needs Apple's own multi-speaker grouping and MFi-style device
authentication, which pulls in a much bigger dependency chain for very little
benefit here: Snapcast is already doing the multi-room fan-out and sync on
the backend, so this receiver only needs to look like *one* AirPlay speaker.
Your phone won't be able to add other AirPlay-2-only speakers to the same
Apple-orchestrated group as this one, but that's not how sync happens in this
setup anyway.

## Setup

1. Add this repository to **Settings → Add-ons → Add-on Store → ⋮ →
   Repositories**, then install **CastSync Receiver**.
2. Set `friendly_name` to what you want the AirPlay speaker / Spotify device
   to be called, and start the add-on.
3. On your phone/Mac: the name should appear in the AirPlay picker and in
   Spotify's device list within a few seconds (mDNS/zeroconf).
4. Install the **Snapcast** integration in Home Assistant core
   (**Settings → Devices & Services → Add Integration → Snapcast**), pointing
   it at your HA box's IP, port `1705`. This is a built-in integration, not
   HACS. It will create a `media_player` per Snapcast group and per client.
5. Put a `snapclient` on each display/speaker you want in sync — a
   Raspberry Pi Zero 2 W + USB or HAT DAC is the usual cheap endpoint:

   ```bash
   sudo apt install snapclient
   # /etc/default/snapclient -> SNAPCLIENT_OPTS="-h <ha-box-ip>"
   sudo systemctl enable --now snapclient
   ```

   Chromecasts and smart TVs **cannot** run snapclient — they're not part of
   this sync domain. That's the [CastSync integration](../ha-castsync)'s job,
   with its own, looser accuracy budget.

## Options

| Option | Default | |
|---|---|---|
| `friendly_name` | `Home Assistant` | Name shown in AirPlay/Spotify pickers |
| `enable_airplay` | `true` | |
| `enable_spotify` | `true` | |
| `airplay_password` | *(empty)* | Optional AirPlay connection password |
| `log_level` | `info` | Passed straight through, `trace` is very noisy |

## Ports

Exposed under `host_network: true` (required for mDNS/SSDP to work at all —
multicast doesn't reliably cross the default Docker bridge):

- `1704` — Snapcast stream port (what `snapclient` connects to)
- `1705` — Snapcast control port (what HA's Snapcast integration connects to)
- `1780` — Snapserver web UI, useful for confirming streams are live
- `5000` — AirPlay RTSP

## Troubleshooting

- **Nothing shows up in AirPlay/Spotify**: check the add-on log for the
  service actually binding; confirm `host_network: true` took effect (some
  HA OS installs on certain hardware restrict host networking — check the
  Supervisor system log if the add-on won't start at all).
- **Audio plays but multiple rooms drift**: that's a `snapclient`-side
  question, not this add-on — check `snapclient -h <ip> -v` logs on the
  lagging device.
- **Web UI at `:1780` shows the stream idle**: the source app (AirPlay/
  Spotify) hasn't connected yet, or picked the wrong stream — each source
  shows as a separate `[stream]` entry.
