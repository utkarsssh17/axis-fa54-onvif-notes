# AXIS FA54 ONVIF source configuration mapping

The FA54 can have different default ONVIF `VideoSourceConfiguration` layouts. An offline comparison of four firmware images found 20 configurations in 9.80.132 and 10.12.363, and 5 in 11.11.238 and 12.11.147. A client that reuses a fixed configuration token can therefore select a different sensor port.

| Firmware inspected | Configurations per source | Total with Quad View enabled |
|---|---:|---:|
| 9.80.132 | 4 | 20 |
| 10.12.363 | 4 | 20 |
| 11.11.238 | 1 | 5 |
| 12.11.147 | 1 | 5 |

These counts come from initialization code and shipped settings, not a live camera capture. They apply to the exact builds listed, with Quad View enabled when configurations are generated. All four images target the FA54 and define four sensor inputs plus a Quad View channel.

## Default mapping

A configuration's `token` identifies the configuration object. Its `SourceToken` selects the video source. Several configurations can select the same source. Both are string identifiers; the numeric values below describe FA54 defaults, not a general ONVIF numbering rule.

| Source | `SourceToken` | VAPIX `camera` | Config tokens in 9.80.132 / 10.12.363 | Config token in 11.11.238 / 12.11.147 |
|---|---|---:|---|---|
| Sensor port 1 | `0` | 1 | `0`–`3` | `0` |
| Sensor port 2 | `1` | 2 | `4`–`7` | `1` |
| Sensor port 3 | `2` | 3 | `8`–`11` | `2` |
| Sensor port 4 | `3` | 4 | `12`–`15` | `3` |
| Quad View | `4` | 5 | `16`–`19` | `4` |

Quad View combines the four inputs; the FA54 has no fifth sensor port. Configuration count is also distinct from the number of media profiles or encoder configurations.

## How a fixed token can select the wrong input

Consider a client that attaches configuration token `"2"` to a profile intended for sensor port 1:

| Default layout | Configuration token | Stored `SourceToken` | Selected input |
|---|---|---|---|
| 9.80.132 / 10.12.363: 20 configurations | `"2"` | `"0"` | Port 1 |
| 11.11.238 / 12.11.147: 5 configurations | `"2"` | `"2"` | Port 3 |

The token exists in both layouts, so attaching it can succeed while selecting the wrong input. If port 3 is unused, a stream from that input may show "No video". This is a failure mode inferred from the mappings; no live-camera reproduction is included here.

Passing `SourceToken="2"` is different: that source corresponds to port 3 in all four inspected builds. `AddVideoSourceConfiguration` takes a **configuration token**, not a source token or a physical port number.

## Why the count changes

In 9.80.132 and 10.12.363, the default initialization path for this FA54 topology divides `Image.MaxViewers` by the number of physical sources plus enabled multiview channels. With the shipped settings, that is `20 / (4 + 1) = 4` configurations per source. The generator assigns consecutive configuration tokens to each source.

In 11.11.238 and 12.11.147, the corresponding path uses one configuration per source. The physical inputs and Quad View remain the same.

`Image.NbrOfConfigs=5` counts logical image channels. It can therefore coexist with 20 ONVIF source configurations. Neither count tells you how many sensors are plugged in.

## Selecting the intended source in a client

For an existing profile, call `GetProfile` or `GetProfiles` and inspect its `VideoSourceConfiguration.token` and `SourceToken`. `GetVideoSourceConfigurations` lists the available configuration objects and their source references.

When assigning a source to a profile:

1. Identify the source for the intended input. For port 1 in the inspected FA54 topology, this is `SourceToken="0"`.
2. Call `GetCompatibleVideoSourceConfigurations` with the target `ProfileToken`.
3. Select a returned configuration with the intended `SourceToken` and suitable bounds. If none matches, report that condition rather than choosing the first entry.
4. Call `AddVideoSourceConfiguration` with the profile token and the selected configuration's returned `token` as `ConfigurationToken`.
5. Read the profile back and verify video from the intended input.

These operations are defined by the [ONVIF Media Service specification](https://www.onvif.org/specs/srv/media/ONVIF-Media-Service-Spec-v1606.pdf), sections 5.2.4 and 5.4. Rebind the profile to a suitable existing configuration when possible. Changing a configuration itself can affect other profiles that share it.

Treat configuration tokens as opaque identifiers. Do not derive a port from a configuration token, its position in the response, or the total configuration count. Refresh the mapping after upgrades or configuration changes. The inspected implementations in 9.80.132, 10.12.363, and 12.11.147 allow changing a configuration's stored `SourceToken`, so even the default table is not a substitute for querying the device.

## Migration and limits

The inspected 9.80.132 and 10.12.363 builds use `media.conf` schema 1.8. The inspected 11.11.238 and 12.11.147 builds use schema 1.9 and contain a migration that updates existing profile references through each old configuration's `SourceToken`, reduces the default layout to tokens `0`–`4`, and assigns those retained configurations matching source tokens. Those schema numbers are separate from firmware release numbers.

That migration does not prevent a client from later reattaching a hardcoded configuration token and selecting a different source. Upgrade behavior and a client's subsequent profile setup are separate paths.

[Migration details](MIGRATION.md) walks through the order of operations, the token `"2"` example before and after migration, shared configurations, and how to distinguish a migration problem from a later client change.

The first release that changed the count has not been identified. Saved settings can differ from generated defaults. No upgrade or client repair was tested on a running FA54.

[Comparison basis and validation limits](METHOD.md) describes how the findings were derived. [firmware.json](firmware.json) identifies the compared images by hash. This repository contains explanatory text and identifying metadata. Firmware is available from Axis under its own terms.
