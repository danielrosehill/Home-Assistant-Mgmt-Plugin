---
name: group-automations
description: Group Home Assistant automations with a label, then turn them on or off as a batch, or add and remove members.
---

# Group automations with a label

Use on "group these automations", "control the X automations as a group", "turn off all the X automations".
A label is the group: automations can carry several, and a label can be the target of any automation action. Group
helpers in the UI don't take automations.

## 1. Load your settings

Find the data folder: `$HOME_ASSISTANT_MGMT_HOME`, `$CLAUDE_USER_DATA/home-assistant-mgmt/`, then
`~/.claude-user-data/home-assistant-mgmt/`. Read `config.json` and `group-automations.md` if they exist; the
notes win over the defaults below. With neither, ask for the API URL and token source, and offer `onboard` at the end.
`$API` below is the REST base ending in `/api`; `$HA_TOKEN` is the resolved token (never print it).

## 2. Choose the members

List automations from `GET $API/states`, match by name, and resolve entity ids; never guess them. If the user names a
theme ("the Shabbat ones"), show what matched. Cover announcements, scenes and light effects, not just obvious names.

## 3. Create the label and tag the members

Labels live in the registries, which are websocket-only (`$HA/api/websocket`):

| Step | Command |
|---|---|
| Existing labels | `config/label_registry/list` |
| Create | `config/label_registry/create` `{"name":"Evening automations","icon":"mdi:weather-night"}` → `label_id` |
| Read a member's labels | `config/entity_registry/get` `{"entity_id":"automation.x"}` |
| Tag | `config/entity_registry/update` `{"entity_id":"automation.x","labels":[...]}` |

- `labels` **replaces** the whole list: send the existing labels plus the new one.
- An automation needs an `id` to be in the registry. Without one it can't be labelled: add an `id` first.
- Reuse an existing label that fits. Creating is not idempotent; check the list first.

## 4. Operate the group

```bash
# off / on, until changed
curl -s -H "Authorization: Bearer $HA_TOKEN" -H 'Content-Type: application/json' \
  -d '{"label_id":"evening_automations"}' $API/services/automation/turn_off    # or turn_on
```

- For a pause that ends by itself, use skill `pause-automations` with the same label.
- `turn_on` enables every member, including one that was off on purpose. Read the states first and say so.
- `turn_off` also stops a run in progress; pass `"stop_actions": false` to let it finish.

## 5. Verify

Read the registry back (members carrying the label) and the state of each member. Report members by name.

## Data storage

Reads `config.json` and `group-automations.md`. Writes nothing there; the group lives in Home Assistant.
