---
name: pause-automations
description: Pause Home Assistant automations between dates with AutoSnooze; they turn back on by themselves. Also extend or end a pause.
---

# Pause automations between dates

Use on "pause these automations until X", "disable X while we're away", "turn off Y between these dates".
Uses the HACS integration [AutoSnooze](https://github.com/mossipcams/autosnooze). Don't build automations for this.

## 1. Load your settings

Find the data folder: `$HOME_ASSISTANT_MGMT_HOME`, `$CLAUDE_USER_DATA/home-assistant-mgmt/`, then
`~/.claude-user-data/home-assistant-mgmt/`. Read `config.json` and `pause-automations.md` if they exist; the
notes win over the defaults below. With neither, ask for the API URL and token source, and offer `onboard` at the end.
`$API` below is the REST base ending in `/api`; `$HA_TOKEN` is the resolved token (never print it).

## 2. Make sure AutoSnooze is installed

`sensor.autosnooze_snoozed_automations` exists → installed. Otherwise install it through HACS, which needs a core
restart (confirm first), then add the integration under Devices & services. Its telemetry is **on by default**: turn it
off in Configure if the user doesn't want it.

## 3. Settle what and when

- **What:** resolve entity ids from `GET $API/states`. A labelled group (skill `group-automations`) goes by label.
- **Until when:** if the user gives an event ("until they're back"), look the date up (calendar) and state it.
- **No end date** ("disable for now"): not AutoSnooze. Use `automation.turn_off` and say it won't come back by itself.

## 4. Pause

```bash
H=(-H "Authorization: Bearer $HA_TOKEN" -H 'Content-Type: application/json')
# now until a date
curl -s "${H[@]}" -d '{"entity_id":["automation.a","automation.b"],"resume_at":"2030-01-15 16:30:00"}' $API/services/autosnooze/pause
# between two dates
curl -s "${H[@]}" -d '{"entity_id":["automation.a"],"disable_at":"2030-01-10 08:00:00","resume_at":"2030-01-15 16:30:00"}' $API/services/autosnooze/pause
# a labelled group
curl -s "${H[@]}" -d '{"label_id":["evening_automations"],"resume_at":"2030-01-15 16:30:00"}' $API/services/autosnooze/pause_by_label
```

| Change | Service |
|---|---|
| End now (turns on) | `autosnooze.cancel` `entity_id`; everything: `cancel_all` |
| Drop one that hasn't started | `autosnooze.cancel_scheduled` `entity_id` |
| Extend or shorten | `autosnooze.adjust` `entity_id` + `days` / `hours` / `minutes` (negative shortens) |
| Change the end date | `pause` or `pause_by_label` again with the new `resume_at`; it replaces the old pause |

Times are local to Home Assistant. Give one way of ending only: `resume_at`, a duration, or a preset.

## 5. Gotchas

- At the end each automation is turned **on**, even if it was off before the pause. Leave out ones that should stay off.
- Names containing words like alarm, siren, smoke, lock, fire or gas need `"confirm": true`. Other safety automations
  aren't detected, so never include one without asking.

## 6. Verify

An HTTP 200 is not proof. Read `sensor.autosnooze_snoozed_automations`: attribute `paused` (or `scheduled`) must list
each automation with the right `resume_at` (UTC), and each paused automation's state must be `off`.

The card `custom:autosnooze-card` lets the user do the same from a dashboard.

## Data storage

Reads `config.json` and `pause-automations.md`. Writes nothing there; pauses live in Home Assistant.
