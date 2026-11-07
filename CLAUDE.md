# switch_light_group_extended — notes for Claude sessions

Home Assistant custom integration (HACS): switch and light group helpers with more state
modes (`last`, majority), a refresh of the members that disagree, and optional
synchronisation of the members (the Multicontrol behaviours). Code in
`custom_components/switch_light_group_extended/`; what it does in `README.md`; how the code is
organized in `ARCHITECTURE.md` (read it before changing any code); user-facing changes in
`CHANGELOG.md` and the README "Version highlights".

One line for now: `main`. Read `BRANCH.md` first. Work branch for pull requests:
`claude/main-port` (from `main`) — always this one, never a new one, whatever name a session
is given; reset to `main` after each merge. Nothing is merged without the user asking.

## The aim — set by the user, not negotiable

A group that does what a group should do: keep its members in the same state, matching the
wanted state, although the members differ; avoid as far as possible any self-sustained
oscillation; and, when the group loses its cohesion, report the state of its members as
sensibly as possible. Designed as a control engineer would: causes (events) kept apart from
consequences (commands), loops open wherever possible, what an event-driven system loses
caught by reading again.

- **A command is sent only on an event** (an unmasked member change, a press, a command on
  the group) — never from the computed group state. The group state is display only.
- **A function that closes a loop** (a member's change sent to the other members, protected
  only by the masking delay) can be disabled completely, is disabled by default, and shows the
  warning "May cause oscillation. In case of problem, try increasing the masking delay." A new
  function is classified open or closed loop before it is coded; a closed one follows this
  rule.
- **Commands carry absolute states** (on / off, values), never a toggle sent to a member: a
  duplicated event changes nothing, a lost one is corrected by the next.
- **The refresh only reads.** It never sends an on/off command.

## STANDARD. EVERYTHING IS STANDARD. — set by the user, not negotiable

- Every option, setting, menu, choice, persisted value and default works **the same way as
  the others of its kind** — same storage, same code path, same rules. Never a variant.
- The integration follows Home Assistant's own patterns, the built-in Group helpers first
  (`group/config_flow.py`, `group/switch.py`, `group/light.py`): what they do, it does the same
  way; it differs only where the README says why.
- **An EXCEPTION must be agreed by the user, after being warned** that it is a special case and
  which rule it breaks. Warn first; never code a special case and leave the user to find it.
- A special case found in the existing code: report it to the user as such, before anything
  else.

## How to work — set by the user, not negotiable

- **Code what is asked, exactly.** A problem detected, something to correct, a case the
  instruction doesn't cover: say it and ask before coding it — never decide it alone, even
  when it looks obvious.
- **An instruction unclear:** reformulate it and wait for the answer — don't interpret.
- **An instruction too summarized** to be coded as is: say so and ask the user to detail it.
- **The full test suite:** only when the user says the modifications are finished, or asks for
  it. Meanwhile, targeted tests only.
- **Never without being asked:** a version bump, a merge into `main`, a PR. Work is committed
  and pushed on the work branch only.
- **A question or a doubt about what is being done** ("why", "who asked for that"), or an
  interruption: answer it first and stop — no code, no test, no commit until the user gives
  the next instruction.
- **After an interruption by the user:** check what was interrupted and clean it up or finish
  it — a file written in part, a scratch file left in the repository, a process still running.
- **What the user observes is a fact, not a hypothesis.** The user sees the devices, Zigbee2MQTT
  and Home Assistant; Claude doesn't. What the user says they saw weighs as much as a
  screenshot ("it works from Zigbee2MQTT" is a fact, not something to check again). An
  observation that contradicts Claude's idea: the idea is wrong until proven otherwise — drop
  it, or ask a question naming the contradiction. "It has nothing to do with it", said once:
  stop that track. The user can be wrong too: then say so plainly, with the fact that
  contradicts them, and let them decide.
- **A text written in the user's name** (an issue, an answer, release notes): built from the
  user's own words; every statement traceable to them or to a result both have seen.
- **Words have a meaning.** "May" is not "must", an example is not a rule, a goal is not an
  absolute rule. Do what is asked, at the scale asked; when the scale is unclear, ask.
- **An example is a symptom.** A case the user gives (a device, an entity, a timing) is never
  the thing to patch: find the general cause, explain it, propose the general rule; the
  example only checks that the rule holds.
- **No patch for a special case.** Clean code and a clean architecture come first, above
  every other goal; an improvement is taken only when it comes from a general rule.
- The user writes in French; answers in French. Repository files are in English.

## Versions — set by the user, not negotiable

- **A release publishes the merged files as they are.** Every merge must be publishable as is:
  version number, CHANGELOG and docs final — no build number, no `[Unreleased]`.
- **The version is the software's and its docs'.** A merge that touches only Claude's own files
  (`CLAUDE.md`, `BRANCH.md`, `ARCHITECTURE.md`) keeps the version as it is.
- **The version number is the last release's + 1.** Before each merge, check on GitHub the last
  release and the dead versions (a tag without a release); the version merged — in
  `manifest.json` and the CHANGELOG — is the last release + 1, past any dead version. Several
  merges can carry it until the user releases it. A released number is never carried again.
- **A dead version** (said by the user): the version goes to the next number everywhere it is
  written, and a new merge is made. A tag without a release seen without the user having said
  so: ask whether that version is dead, before merging.
- **The CHANGELOG's section is the version's number** (`## v0.1.0`), never `[Unreleased]`.
- **Build numbers** (`0.1.0b1`…) only in Claude's working copy and in discussions — never
  merged: not in `manifest.json`, the CHANGELOG, the docs, a code comment, a commit or PR title.

## A README is not a changelog — set by the user, not negotiable

| Document | What it says |
|---|---|
| README.md | How to use the integration **as it is now**. Never what changed or when: no version number, no "since", "now", "no longer". |
| "📋 Version highlights" (README.md, before the table of contents, formatted as lowpass_dt's — the only place naming versions) | One line per important feature, at the one version where it first worked and could be used. A version of fixes and refinements only has no line. |
| CHANGELOG.md (and the release notes) | Everything, version by version. |

Until the first release, the README describes the planned behaviour and says so in its status
line. Each paragraph goes in the section of its subject; after a README edit, check every link
and table-of-contents entry.

## Shared logic with Multicontrol

[Cook23/Multicontrol](https://github.com/Cook23/Multicontrol) (Home Assistant blueprints) is a
different project by the same author. The synchronisation modes of this integration are its
three blueprints (`multicontrol`, `slave`, `hybrid`), and the README says so. A change to these
rules here must be mirrored there (and vice versa), or the docs become wrong; tell the user
when a change here needs a matching change there.

Rule set by the user: each session manages its own repository's code. A session may write
`.md` files in another of the user's repositories only with the user's explicit approval, and
never modifies another repository's code.

Must stay identical:

- **Masking.** The first change that is not masked acts immediately; every later change
  re-arms the masking delay (1 s by default) and is ignored until the delay has passed without
  any change.
- **`hybrid`.** A press is never masked but starts the masking delay; it toggles the internal
  state, sent to the relays only (the slave relays follow through their Zigbee bindings);
  after the masking delay, the slave relays are forced to the internal state; the state of the
  slave relays is never read by the synchronisation.

Known, deliberate differences: the internal state is kept by the group (no `input_boolean`);
the propagation of relay changes is a separate option of `hybrid`, off by default.

## Related project: lowpass_dt, history-explorer-card

[Cook23/lowpass_dt](https://github.com/Cook23/lowpass_dt) and
[Cook23/history-explorer-card](https://github.com/Cook23/history-explorer-card) share no logic
with this integration; their `CLAUDE.md` are where these rules come from.
