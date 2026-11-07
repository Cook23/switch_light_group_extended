# Switch & Light Group Extended

Home Assistant helper that creates switch and light groups with more ways to compute the group state: **last change** (the group takes the on/off state of the member that changed last) and **majority** (the group takes the state of most of its members).

Optionally, the group keeps its members in sync by itself (n-way switching, from the [Multicontrol](https://github.com/Cook23/Multicontrol) blueprints).

The aim is a group that does what a group should do: keep its members in the same state, matching the wanted state, although the members differ (lost reports, Zigbee bindings, several sources of commands), while avoiding as far as possible any self-sustained oscillation; and, when the group loses its cohesion, report the state of its members as sensibly as possible (state modes).

> Status: planned, not usable yet.

## Why

The built-in [Group](https://www.home-assistant.io/integrations/group/) integration computes the state of a switch or light group from the current states of its members:

- `any` (default): on if at least one member is on;
- `all`: on only if all members are on.

For members that are meant to be in the same state (n-way switches, switches kept in sync by Zigbee bindings, several relays driving the same load), neither is the state of the group:

- one member that did not report its new state (lost report, see [romasku/tuya-zigbee-switch#548](https://github.com/romasku/tuya-zigbee-switch/issues/548) and [#74](https://github.com/romasku/tuya-zigbee-switch/issues/74)) keeps the group on (`any`) or off (`all`);
- the group state then cannot be trusted, and only the group commands stay useful.

In `last` mode, the group follows the last state change received from any member. With switches synchronised by Zigbee bindings, the bound switches always report the new state, even when the pressed switch's own report is lost, so the group shows the right state.

In `majority` modes, one stale member is outvoted by the others as soon as the group has three members or more.

The same request exists on the community forum: [Add "last changed sub-item" for group state](https://community.home-assistant.io/t/add-last-changed-sub-item-for-group-state/453758).

## Planned behaviour

Group types: `switch` and `light`.

Modes:

| Mode | Group state |
|---|---|
| `any` | on if at least one member is on (same as the built-in group) |
| `all` | on only if all members are on (same as the built-in group) |
| `last` | state of the last member change to on or off |
| `majority_on` | state of the majority of members; on in case of a tie |
| `majority_off` | state of the majority of members; off in case of a tie |
| `majority_last` | state of the majority of members; `last` state in case of a tie |

In `majority` modes, only members that are `on` or `off` are counted; `unavailable` and `unknown` members are left out.

In `last` mode, and for ties in `majority_last` mode:

- only changes to `on` or `off` are taken into account;
- a change from `unavailable` or `unknown` (member coming back after a restart or a loss of connection) is ignored;
- the state is restored when Home Assistant restarts;
- if no change was received yet, the group starts from the `any` state.

In every mode, `turn_on` / `turn_off` / `toggle` on the group are sent to all members, as with the built-in group.

Option **refresh members that disagree** (off by default, any mode): when a member stays in a state different from the group state, the group asks it to report its state again, to catch a lost report.

- it runs after a delay (setting, a few seconds by default) without any new member change, so that members still switching are not polled; with a synchronisation mode, the delay starts after the end of the masking delay;
- each disagreeing member is polled once per group state change, to avoid loops with a member that does not answer or keeps a different state;
- the refresh method depends on the member, with nothing to configure:
  - ZHA and other polling integrations: `homeassistant.update_entity`, which reads the state from the device;
  - Zigbee2MQTT: MQTT entities ignore `update_entity`, so the group publishes `{"state": ""}` on `zigbee2mqtt/<device>/get` (`{"state_l1": ""}` for a multi-relay device, the endpoint being taken from the entity's unique ID, e.g. `0x…_switch_l1_zigbee2mqtt`). The device is addressed by its IEEE address, read from the Home Assistant device registry, so renaming it in Zigbee2MQTT does not break the refresh. Zigbee2MQTT then reads the attribute from the device and publishes the real state;
- the member's answer then updates the group state as any other change.

For light groups, a second setting chooses how brightness and colour (colour mode, colour, colour temperature, effect) are computed from the members that are on. Its default, `same as state`, follows the state mode:

| State mode | Brightness and colour with `same as state` |
|---|---|
| `any` | `highest` |
| `all` | `lowest` |
| `last` | `last` |
| `majority_on`, `majority_off`, `majority_last` | `mean` |

It can be set to another value to override this:

| Brightness and colour | Values |
|---|---|
| `highest` | highest brightness; colour of that member |
| `lowest` | lowest brightness; colour of that member |
| `last` | values of the last member that changed |
| `mean` | mean, as the built-in light group |

With `last`, a change of brightness or colour of a member that is on counts as a change of that member; the values are restored when Home Assistant restarts.

### Synchronisation (Multicontrol)

Option **synchronisation** (`none` by default): the group keeps its members in sync by itself, without loops, with the three behaviours of the [Multicontrol](https://github.com/Cook23/Multicontrol) blueprints, so that n-way switching from several sources (physical switches, Zigbee bindings, Home Assistant) needs no automation.

| Synchronisation | Behaviour |
|---|---|
| `none` | the group only computes its state and forwards its commands (built-in behaviour) |
| `multicontrol` | every member is both source and target: the first unmasked change of a member to on or off is sent to all the other members |
| `slave` | members are split into relays and slave relays (relays synchronised among themselves by native Zigbee bindings); the first unmasked change of a relay is sent to the other relays and to the slave relays; slave relays trigger nothing |
| `hybrid` | the relays and slave relays of `slave`, plus the "switch press action" entities of switches running the [romasku/tuya-zigbee-switch](https://github.com/romasku/tuya-zigbee-switch) firmware: a press toggles the internal state and is sent to the relays (the slave relays follow through their bindings); after the masking delay, the slave relays are forced to the internal state (anti-desynchronisation); the propagation of relay changes of `slave` is a separate option |

Common rules, as in Multicontrol:

- masking: the first change that is not masked acts immediately; every later change re-arms the masking delay (setting, 1 s by default) and is ignored until the delay has passed without any change, so the echoes of the group's own commands are ignored;
- in `hybrid` mode, a press is never masked but starts the masking delay; the internal state is kept by the group itself (restored at restart), no `input_boolean` helper is needed;
- the commands sent to the members follow the state of the source member (on/off, and brightness and colour for light groups).

Oscillation: the functions are split in two kinds.

Open loop, no oscillation possible, always available:

- commands on the group (`turn_on`, `turn_off`, `toggle`): sent to all members, they set the internal state and start the masking delay;
- in `hybrid` mode, presses: a press is an event of the button, not of a relay, and its commands never come back as a press;
- in `hybrid` mode, the anti-desynchronisation: it forces the slave relays to the internal state, and the state of the slave relays is never read;
- the refresh of disagreeing members: it only reads states, it never sends on/off commands;
- the group state shown in Home Assistant (state mode): display only, it never feeds back into the synchronisation. The internal state of `hybrid` mode is separate from it and never set from the member states, so a stale report cannot be copied to the other members.

Closed loop, protected only by the masking delay: the propagation of member changes to the other members (a member is both source and target). This covers the `multicontrol` and `slave` modes and the "propagate relay changes" option of `hybrid` mode. These functions can be disabled completely and are disabled by default (`none` synchronisation, option off in `hybrid`). Choosing one of them shows a warning in the config flow: "May cause oscillation. In case of problem, try increasing the masking delay."

To decide in use: whether slave relays count in the state mode in `slave` and `hybrid` modes (Multicontrol never reads their state; the `last` mode relies on their reports).

## Installation (planned)

Through [HACS](https://hacs.xyz/), as a custom repository:

1. HACS → Integrations → ⋮ → Custom repositories → add this repository, category "Integration".
2. Install "Switch & Light Group Extended" and restart Home Assistant.
3. Settings → Devices & services → Helpers → Create helper → "Switch & Light Group Extended".

## Design

Modelled on the built-in [Group](https://www.home-assistant.io/integrations/group/) helpers of Home Assistant, as a separate integration with its own domain (`switch_light_group_extended`). The built-in group integration is left untouched.

- Config flow: starts from the built-in `group/config_flow.py`. An options flow edits the group after creation. Steps:
  1. menu: group type, switch or light (as the built-in group);
  2. menu: "New group" or "Import from an existing group";
  3. import only: pick an existing Home Assistant group of the same type (UI or YAML); its members are read from its `entity_id` attribute;
  4. form: name, members, state mode and, for light groups, brightness and colour (default `same as state`), the refresh option and its delay, the synchronisation mode and its masking delay, with the members prefilled when importing;
  5. `slave` and `hybrid` synchronisation only: slave relays and, for `hybrid`, the switch press action entities and the "propagate relay changes" option (off by default). The oscillation warning is shown for `multicontrol`, `slave` and that option.

  The imported group is left untouched.
- Entities: start from the built-in `group/switch.py` and `group/light.py` (`GroupEntity`, `SwitchGroup`, `LightGroup`). The `all` checkbox becomes a choice of the six state modes, light groups get the brightness and colour setting, and state restore is added for `last` and `majority_last`. `hide_members` is kept.

Like [Climate Group Helper](https://github.com/bjrnptrsn/climate_group_helper), it ships as a custom integration installable through HACS.

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
