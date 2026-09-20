# CastSync Add-ons

Home Assistant add-on repository. Currently one add-on:

- **[castsync_receiver](castsync_receiver/DOCS.md)** — AirPlay + Spotify
  Connect receiver feeding a Snapcast server, for tight-sync multi-room audio.

This is the receiver half of a two-repo setup. The other half, the
[CastSync integration](../ha-castsync) (install via HACS), fans video out to
Chromecasts and smart TVs with looser, seek-based sync — a different problem
with a different accuracy ceiling. See that repo's README for why they're
split rather than one system: audio sync needs a shared clock (Snapcast
provides one, but only for devices that run `snapclient`); Chromecasts and
smart TVs can't join that clock domain, so they get the fan-out-and-seek
approach instead.

## Install

**Settings → Add-ons → Add-on Store → ⋮ (top right) → Repositories**, add:

```
https://github.com/audio948/ha-castsync-addons
```

This repo is **private**, so the Supervisor's add-on store needs a token embedded in
that URL to read it (`https://<token>@github.com/audio948/ha-castsync-addons`) — see
the installer walkthrough for how to generate and use that token.

Then install **CastSync Receiver** from the store list that appears.

## Status

Unbuilt — see [castsync_receiver/DOCS.md](castsync_receiver/DOCS.md#status-unbuilt)
for exactly what that means and what to do if the first build fails.
