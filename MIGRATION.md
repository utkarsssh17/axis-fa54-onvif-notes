# How the source configuration migration works

The inspected 11.11.238 and 12.11.147 builds contain a migration that consolidates multiple configurations per source into one. It updates existing profiles before changing the configurations they reference, so a profile can continue selecting the same input even though its configuration token changes.

This description comes from offline examination of the firmware implementation. No live upgrade was tested. The examples assume the default four sensor inputs and enabled Quad View, giving five logical sources.

## What triggers it

The migration checks the saved media configuration format. The inspected 10.12.363 build uses schema 1.8; the two newer builds use 1.9. These are configuration-format versions, separate from firmware release numbers.

Schema 1.8 enters the consolidation step directly. Earlier formats pass through preceding migration steps. A configuration already marked 1.9 skips this consolidation; a later client change to a profile does not cause it to run again. The first firmware release introducing this change remains unknown.

## Order of operations

1. **Rewrite every existing profile's source-configuration reference.** Read the profile's old configuration token, look up that configuration's stored `SourceToken`, and use that source token as the profile's new configuration reference. All profiles are processed before source configurations are changed or removed.
2. **Normalize or remove each source configuration.** With five logical sources, keep configuration tokens `"0"` through `"4"` and set each retained configuration's `SourceToken` to match its token. Remove the remaining configurations: in the default 20-entry layout, these are tokens `"5"` through `"19"`.
3. **Save the migrated configuration.** The profile updates and configuration changes are completed in memory before the final save.

The profile update follows the old stored `SourceToken`; it does not calculate a source by dividing the configuration token. This matters when a configuration's source assignment was customized.

The retained objects also change meaning. Old configuration `"2"` survives, but its default source changes from `"0"` to `"2"`. Profiles formerly using it have already been redirected to configuration `"0"`.

## A profile using token `"2"`

For a profile intended to stream sensor port 1, the expected sequence is:

| Stage | Profile's configuration token | Configuration's `SourceToken` | Input |
|---|---|---|---|
| Before migration | `"2"` | `"0"` | Port 1 |
| After migration | `"0"` | `"0"` | Port 1 |
| Client later attaches token `"2"` again | `"2"` | `"2"` | Port 3 |

The last step is a separate client action. The migration can preserve the profile's input and a subsequent provisioning request can still select another input. If port 3 is unused, that sequence could explain a "No video" picture. It has not been reproduced on a running device here.

## Settings and failure handling

Profiles that previously selected different configurations for the same source now share one configuration. Later changes to that shared object can affect every profile using it.

The consolidation includes limited rotation transfer into the retained source-numbered configurations. It does not establish preservation of every per-configuration customization, including different bounds or names. Avoid treating it as a complete merge of all old settings.

Errors reading profile references or source-configuration fields, or removing a configuration group, lead to failure before this routine's final save. The changes are prepared in memory. This does not establish crash-safe rollback, successful recovery after a failed save, or what the calling service does after an error.

## Checking migration separately from client setup

On a test device, pause the provisioning client before upgrading. Record the firmware version, connected ports, Quad View state, profiles, configuration tokens, and source tokens. After upgrading, inspect the same profiles and verify their video before allowing the client to run again. Then run the client and compare the references and video a second time.

That sequence separates the firmware's migration from later profile changes. A fresh configuration can also have five entries without ever passing through this migration, so the count alone cannot establish a device's history or explain a missing picture.

See the [client selection procedure](README.md#selecting-the-intended-source-in-a-client) for selecting configurations by their returned source references.
