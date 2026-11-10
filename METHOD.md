# Comparison basis and validation limits

Analysis date: 2026-11-10.

The question was whether FA54 firmware changes the mapping between ONVIF video source configuration tokens and sensor inputs. The intended use is client interoperability: selecting the same physical input across firmware versions without relying on a fixed configuration identifier.

This comparison used official FA54 firmware images from the [Axis support page](https://www.axis.com/products/axis-fa54-main-unit/support). Each image's SHA-256 matched the value published there. The hashes identify the inputs; this was not an independent check of firmware signatures.

## Inputs

| Version | Image | SHA-256 |
|---|---|---|
| 10.12.363 | `FA54_10_12_363.bin` | `ba93d822cb1b9ac12a015cfea10304dd251993fb9b065f6c2c40ba335d4d5292` |
| 11.11.238 | `FA54_11_11_238.bin` | `a8b59184218f51fe995de6774c35359a087e074307f986ab342014f6d5d32da7` |
| 12.11.147 | `FA54_12_11_147.bin` | `594d75ea67d7f3ed04181895e9d899771c4ba47fd96233f7096079d2d92562fa` |

[firmware.json](firmware.json) records the vendor download URLs, image sizes, hashes, and expected default counts.

## Basis of the comparison

The version-specific findings were inferred from offline examination of the implementation and default settings in the downloaded firmware. No firmware executable was run, and no live ONVIF response was captured. This write-up summarizes the resulting behavior; it does not include vendor code.

The comparison covered:

- The default number of configurations created for each logical source.
- The assignment of configuration tokens and source tokens.
- How a profile's selected configuration resolves to an input channel.
- Whether a configuration can refer to a different source after creation.
- How existing profile references are updated when the saved configuration format changes.

For the shipped FA54 settings, 10.12.363 allocates four configurations to each of five logical sources. The other two builds allocate one per source. Configuration tokens and source tokens are separate identifiers, even when their values happen to match.

Channel selection follows the configuration's stored `SourceToken`. In the inspected FA54 topology, adding one to that source index gives the VAPIX camera number. This relationship does not apply to the configuration token itself. The ability to change a configuration's source was checked in 10.12.363 and 12.11.147.

The newer builds also contain a migration from media configuration schema 1.8 to 1.9. It resolves old profile references through their source tokens, updates the references, and removes surplus source configurations. This explains how the default 20-entry layout can become the five-entry layout during migration. A running-device upgrade was not tested.

The [migration walkthrough](MIGRATION.md) describes the observed ordering, limited setting transfer, and error paths. Its before-and-after example is a prediction from the implementation, not a captured device sequence.

## Connecting logical sources to ports

The firmware metadata identifies FA54 hardware ID 734 in all three images. The model's channel definitions are byte-identical. They map channels 1–4 to inputs 1–4, and channel 5 to a composite of those four inputs.

The default video-service settings in 10.12.363 and 11.11.238 assign sources 0–3 to channels 1–4 and Quad View to channel 5. In 12.11.147, the settings define four source channels and one composite channel. The legacy parameter mapping connects `Image.I0` through `Image.I4` to channels 1 through 5.

These settings connect ONVIF source selection to the device's input topology. A configuration-object count does not establish how many sensors are connected.

## What still needs checking

- No ONVIF response or upgrade was observed on a running FA54.
- Only the three listed builds were inspected. The first release with the new count remains unknown.
- Quad View is checked for availability during configuration creation. The totals assume it is enabled at that point; this does not establish what happens when it is toggled later.
- Saved settings can differ from the generated defaults.
- The fixed-token example is inferred from the default mappings. It is not a recorded outage, and the suggested client procedure has not been tested against a running device.

To check the result on a device:

1. Record its exact firmware version, connected sensor ports, Quad View state, and whether settings were freshly initialized or migrated.
2. Call `GetVideoSourceConfigurations` and record each configuration's `token`, `SourceToken`, and bounds. Record the affected profile's current source configuration with `GetProfile` or `GetProfiles`.
3. Compare the stored source reference with the intended physical input. The count alone cannot establish the cause of a missing picture.
4. On a test profile, query compatible source configurations, attach one for the intended source, and verify both the returned profile and the resulting video.

An upgrade test would separately check whether existing profile references migrate correctly. A profile-creation test would check whether the client discovers the source configuration again. These are different behaviors; the static comparison does not establish either at runtime.

## References

- [Axis Image API](https://developer.axis.com/vapix/network-video/parameter-management/image-api/): logical channels, source indices, and the `camera` parameter.
- [Axis channel lookup guide](https://developer.axis.com/video-streaming-and-recording/video-streaming/how-to-guides/lookup-channels/): interpreting `Image.NbrOfConfigs`.
- [FA54 manual](https://help.axis.com/en-us/axis-fa54-main-unit): hardware and sensor connections.
- [ONVIF Media Service specification](https://www.onvif.org/specs/srv/media/ONVIF-Media-Service-Spec-v1606.pdf): configuration objects and source references.

These references explain the interfaces. The firmware comparison supplies the version-specific count and mapping findings.

## Repository scope

This repository contains original explanatory text and identifying metadata. It includes no vendor software, device captures, credentials, or deployment records.

This is an independent interoperability note. It is not affiliated with or endorsed by Axis. Axis software remains subject to its [license agreements](https://www.axis.com/legal/license-agreements).
