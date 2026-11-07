# Switch & Light Group Extended

Home Assistant helper that creates switch and light groups with one more way to compute the group state: **last change**. The group takes the on/off state of the member that changed last.

> Status: planned, not usable yet.

## Why

The built-in [Group](https://www.home-assistant.io/integrations/group/) integration computes the state of a switch or light group from the current states of its members:

- `any` (default): on if at least one member is on;
- `all`: on only if all members are on.

For members that are meant to be in the same state (n-way switches, switches kept in sync by Zigbee bindings, several relays driving the same load), neither is the state of the group:

- one member that did not report its new state (lost report, see [romasku/tuya-zigbee-switch#548](https://github.com/romasku/tuya-zigbee-switch/issues/548) and [#74](https://github.com/romasku/tuya-zigbee-switch/issues/74)) keeps the group on (`any`) or off (`all`);
- the group state then cannot be trusted, and only the group commands stay useful.

In `last` mode, the group follows the last state change received from any member. With switches synchronised by Zigbee bindings, the bound switches always report the new state, even when the pressed switch's own report is lost, so the group shows the right state.

The same request exists on the community forum: [Add "last changed sub-item" for group state](https://community.home-assistant.io/t/add-last-changed-sub-item-for-group-state/453758).

## Planned behaviour

Group types: `switch` and `light`.

Modes:

| Mode | Group state |
|---|---|
| `any` | on if at least one member is on (same as the built-in group) |
| `all` | on only if all members are on (same as the built-in group) |
| `last` | state of the last member change to on or off |

In `last` mode:

- only changes to `on` or `off` are taken into account;
- a change from `unavailable` or `unknown` (member coming back after a restart or a loss of connection) is ignored;
- the state is restored when Home Assistant restarts;
- if no change was received yet, the group starts from the `any` state.

In every mode, `turn_on` / `turn_off` / `toggle` on the group are sent to all members, as with the built-in group.

Open question for light groups in `last` mode: whether brightness and colour follow the last changed member too, or only on/off.

## Installation (planned)

Through [HACS](https://hacs.xyz/), as a custom repository:

1. HACS → Integrations → ⋮ → Custom repositories → add this repository, category "Integration".
2. Install "Switch & Light Group Extended" and restart Home Assistant.
3. Settings → Devices & services → Helpers → Create helper → "Switch & Light Group Extended".

## Design

Built on the same principle as [Climate Group Helper](https://github.com/bjrnptrsn/climate_group_helper): a separate integration with its own domain (`switch_light_group_extended`), created as a helper from the UI (config flow). The built-in group integration is left untouched.

The entity code starts from the built-in `group/switch.py` and `group/light.py` of Home Assistant, with the `last` mode and state restore added.

Planned layout:

```
custom_components/switch_light_group_extended/
  manifest.json      integration_type: helper, config_flow: true
  __init__.py
  config_flow.py     group type, members, mode
  const.py
  switch.py
  light.py
  translations/en.json, fr.json
hacs.json
```

## Related

- [Multicontrol](https://github.com/Cook23/Multicontrol): Home Assistant blueprints that keep relays in sync without oscillation, and native Zigbee synchronisation with the alternative firmware.

## License

Apache License 2.0, as the Home Assistant code this integration is derived from.
