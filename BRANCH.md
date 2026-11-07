# Branch: main

The only line for now: the integration's first version, not released yet.

How the code is organized, and the rules that keep its parts apart: see
[ARCHITECTURE.md](ARCHITECTURE.md).

## The constraint: clean code

Every change, features and fixes alike, keeps the code clean:
- each piece of code in its own part, as ARCHITECTURE.md sets them;
- one implementation per behavior, shared by every caller (switch and light, every mode):
  no copy, no parallel path;
- the built-in Group code kept recognizable where it is reused, so that its fixes can be
  followed;
- dead code removed, comments saying why rather than history, tests for every behavior.
A change that would break this is redesigned, not merged.

## Planned

The behaviour described in README.md, "Planned behaviour".

## To decide in use

- Whether slave relays count in the state mode in `slave` and `hybrid` synchronisation.
