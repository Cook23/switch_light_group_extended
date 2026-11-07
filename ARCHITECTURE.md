# Architecture

How the integration's code is organized, what each part is responsible for, and the rules
that keep them apart. Written before the code: the code follows it; a part that turns out
wrong is changed here first, with the user's agreement.

## The parts

`custom_components/switch_light_group_extended/`:

| Part | Responsible for | Knows Home Assistant? |
|---|---|---|
| `const.py` | the domain, the names of the options, their values (the state modes, the brightness and colour modes, the synchronisation modes) and their defaults — every list and every default, once | no |
| `state.py` | the state computation: from the members' states and the history of their changes, the group state (`any`, `all`, `last`, the majorities) and the brightness and colour (`highest`, `lowest`, `last`, `mean`) — pure functions | no |
| `sync.py` | the synchronisation: masking, the internal state, which commands to send to which members on an event — a state machine fed with events and the time, returning commands | no |
| `refresh.py` | the refresh of the members that disagree: which member, how (`update_entity`, the Zigbee2MQTT `get` request found from the device registry), when | yes |
| `entity.py` | the group entity common to switch and light: listens to the members, feeds `state.py` and `sync.py`, sends their commands, restores the state, starts the refresh | yes |
| `switch.py`, `light.py` | the platforms: what is specific to a switch or a light group, from the built-in `group/switch.py`, `group/light.py` | yes |
| `config_flow.py` | the config and options flows, from the built-in `group/config_flow.py`: the menus, the import of members from an existing group, the forms, the oscillation warning | yes |
| `__init__.py` | the set-up and unload of a config entry | yes |

## The rules

1. **Decisions are pure, effects are at the edge.** `state.py` and `sync.py` never import
   Home Assistant, never read the clock themselves, never send anything: they take states,
   events and the time, and return a state or a list of commands. `entity.py` and
   `refresh.py` do the effects. So every decision is tested without Home Assistant.
2. **A command comes only from an event.** `sync.py` returns commands only in answer to an
   event (an unmasked member change, a press, a command on the group, the end of the masking
   delay for the anti-desynchronisation). The group state computed by `state.py` is never an
   input of `sync.py`.
3. **Open or closed loop is written in the code.** Each function of `sync.py` that sends a
   member's change to the other members is marked as closed loop, is enabled only by its
   option (off by default, `const.py`), and the config flow shows the oscillation warning for
   it. Checked by a test.
4. **One implementation per thing.** A behavior needed in two places is one function both
   call (switch and light, every mode). A list of names is written once, as a named constant in
   `const.py`.
5. **Every default is written once**, in `const.py`, and applied where the options are read
   (the config flow and the entity's set-up) — never where it is used. A value that depends on
   another option (`same as state` for brightness and colour) is resolved where the options are
   read too.
6. **An option is put in its one form where it is read**; the rest of the code tests only that
   form.
7. **Each part talks to another only through its public functions**; a name starting with `_`
   is its own.
8. **The built-in Group code is reused as is where it fits**, kept recognizable, with its
   license (Apache 2.0); what differs is in its own function, not mixed into the copied code.

## Where a change goes

| To change... | Look in |
|---|---|
| A state mode, a brightness and colour mode | `state.py`, its name and default in `const.py` |
| The synchronisation, the masking | `sync.py` (and Multicontrol, see `CLAUDE.md`) |
| How a member is read again | `refresh.py` |
| What the group listens to, restores, sends | `entity.py` |
| What is specific to a switch or a light | `switch.py`, `light.py` |
| A menu, a form, a warning | `config_flow.py`, `translations/` |
| An option's name, values, default | `const.py` |

## Tests

`pytest` with `pytest-homeassistant-custom-component`, as lowpass_dt:

- `state.py` and `sync.py` on their own, no Home Assistant: every mode, every tie, masking
  timings, the open/closed loop marking;
- the integration against a real Home Assistant core: config flow, entities, restore, refresh,
  synchronisation.
