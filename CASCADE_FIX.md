# Cascade / multi-zone Gree support (uncommitted local patch)

Local working-tree changes to support **central/cascade Gree systems** (one wifi
gateway proxying several indoor heads) and to fix the North American broker host.
Not committed — this file documents the diff so it can be reconstructed or turned
into a PR later.

## The problem

Out of the box this integration assumes a flat topology: **one wifi module = one
AC**. It addresses every device on *its own* MQTT topic (`request/<deviceMac>`).

On a cascade/multi-zone system that assumption breaks. The Gree cloud returns,
per home, a mix of:

- **gateways** — the wifi modules actually connected to the cloud broker. Real
  Gree OUI MAC `58:0D:0D…`, 12 hex chars, e.g. `580d0d37023e`.
- **heads** — the individual indoor units behind a gateway. 14-char MACs ending
  in `00`, e.g. `9abbc91e000000`. They have **no independent cloud/network
  presence**; all their traffic is relayed through their gateway.

All devices sharing a gateway also share that gateway's **encryption key**.

The existing dedup (`greeclimate.cloud_api._filter_duplicate_devices`) keeps the
heads and drops the gateways, then `CloudDevice._detect_parent_mac` derives each
head's parent as `headMac[:-2]` — a topic nobody is subscribed to. Result: every
status poll publishes into the void and times out; no entity ever gets state.

Verified on a real system (3 gateways → 9 heads): when you instead subscribe /
publish on the **gateway** topic, head state immediately streams in on
`status/<gatewayMac>/<headMac>`, and commands sent to `request/<gatewayMac>` with
`tcid=<headMac>` control the right head (this is exactly what the Gree+ app does).

## Changes in THIS repo (`homeassistant-gree-cloud`)

- **`custom_components/gree_cloud/const.py`** — `GREE_MQTT_SERVERS["North American"]`
  was `mqtt-us.gree.com`, which does **not** resolve (NXDOMAIN). Corrected to
  `mqtt-na.gree.com`. Without this, North American accounts fail to connect at all
  with `[Errno -2] Name does not resolve`.

- **`custom_components/gree_cloud/coordinator.py`**
  - In the device-discovery loop, before `device.bind()`: if the
    `CloudDeviceInfo` carries a `parent_mac` (set by the library patch below),
    override `device._parent_mac = parent_mac` and set `device._is_cascade = True`.
    This points all of the head's MQTT topics (subscribe + publish) at its gateway.
  - Downgraded the per-cycle `"Timeout waiting for state update"` log from
    `warning` to `debug`. On cascade systems the gateway pushes state on `status/`
    rather than acking the status *request*, so state is applied via the
    unsolicited path and the request legitimately "times out" every cycle — it is
    expected noise, not an error.

## Companion changes (separate repo: `greeclimate`)

The library half of the fix lives in the **greeclimate** dependency
(pinned `@1.0.3` in `manifest.json`), see `../greeclimate/CASCADE_FIX.md`:

- `CloudDeviceInfo` gains a `parent_mac` field.
- `GreeCloudApi.get_all_devices()` builds a `key → gateway MAC` map (gateway =
  the non-`00` member of each key group) and stamps `parent_mac` onto each head.
- `CloudDevice._handle_mqtt_message()` gains a child-scoped routing guard: when
  `_is_cascade`, only accept `status/`/`response/` messages whose topic's 3rd
  segment equals this head's child MAC — otherwise heads behind the same gateway
  (which share a key + parent topic) would cross-apply each other's state.

## How to turn this into PRs

Two PRs, library first (the integration change reads the new `parent_mac` field):

1. `greeclimate`: the `cloud_api.py` + `cloud_device.py` changes.
2. `homeassistant-gree-cloud`: the `const.py` + `coordinator.py` changes, bumping
   the pinned greeclimate ref once (1) is released.

`origin` = your fork (GTez), `upstream` = davo22. `master`/`main` were identical
to the deployed `1.0.3`, so the diffs apply cleanly.

## Deployment note

The live Home Assistant (personal Proxmox LXC 1032, `172.16.1.32`) runs these
changes as **direct edits** to its installed copies — the integration files under
`/config/custom_components/gree_cloud/` and the library under
`site-packages/greeclimate/`. The library edits do **not** survive a container
image rebuild or an integration reinstall and must be re-applied. These forks are
the source-controlled record of those edits.
