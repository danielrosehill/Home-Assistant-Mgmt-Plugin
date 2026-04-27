---
name: home-assistant-ops
description: Use when the user wants to work on Home Assistant — automations, entities, service calls, TTS, devices — via SSH or the Home Assistant REST API. Reads connection details from `$CLAUDE_USER_DATA/home-assistant-mgmt/config.json` (populated by the `onboard` skill in this plugin). Triggers on phrases like "home assistant", "HA ops", "ha automation", "check home assistant", "call HA service".
---

# home-assistant-ops

Operate against a Home Assistant instance via REST API and (optionally) SSH. All host- and credential-specific values come from the plugin's config — never hard-code them.

## Pre-flight

Resolve the plugin data directory (`${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/home-assistant-mgmt/`) and load `config.json`. If it doesn't exist or is incomplete, tell the user to run the `onboard` skill first and stop.

Available fields after load: `host`, `install_type`, `ssh.{enabled,user,port,key_path}`, `api_url`, `api_token_ref`, `tts_default_target`, `config_path`.

Resolve the bearer token from `api_token_ref` at runtime (1Password / env / file). Don't log it.

## REST API patterns

```bash
TOKEN=<resolved from api_token_ref>
H_AUTH="Authorization: Bearer $TOKEN"
H_JSON="Content-Type: application/json"

# Health check
curl -s -H "$H_AUTH" "$API_URL/api/"

# All entity states
curl -s -H "$H_AUTH" "$API_URL/api/states" | jq '.[] | {entity_id, state}' | head -40

# Single entity
curl -s -H "$H_AUTH" "$API_URL/api/states/<entity_id>"

# Call a service
curl -s -X POST -H "$H_AUTH" -H "$H_JSON" \
  -d '{"entity_id":"<entity_id>", "...": "..."}' \
  "$API_URL/api/services/<domain>/<service>"

# Validate config (HAOS/Supervised only)
curl -s -X POST -H "$H_AUTH" "$API_URL/api/config/core/check_config"

# Restart HA core (DESTRUCTIVE — confirm with user first)
curl -s -X POST -H "$H_AUTH" "$API_URL/api/services/homeassistant/restart" -d '{}'
```

## SSH patterns (when `ssh.enabled`)

```bash
SSH_OPTS=( -p "$SSH_PORT" )
[ -n "$SSH_KEY_PATH" ] && SSH_OPTS+=( -i "$SSH_KEY_PATH" )
ssh "${SSH_OPTS[@]}" "$SSH_USER@$HOST" "<command>"
```

Useful commands once SSH'd in:

```bash
# HAOS / Supervised
ha core info
ha core check
ha core logs --tail 100
ha core restart      # Destructive — confirm first

# All install types
cat $CONFIG_PATH/.HA_VERSION
tail -100 $CONFIG_PATH/home-assistant.log
ls $CONFIG_PATH/automations.yaml $CONFIG_PATH/scripts.yaml $CONFIG_PATH/scenes.yaml 2>/dev/null
```

## Typical tasks

- **Automation authoring/debugging** — read `$CONFIG_PATH/automations.yaml`, edit, validate, restart on confirmation.
- **Entity inspection** — `GET /api/states/<entity_id>` for live state + attributes.
- **Service calls** — `POST /api/services/<domain>/<service>` for one-shot actions.
- **TTS testing** — call `tts.speak` (or the legacy `tts.<provider>_say`) against `tts_default_target` from config. Example:
  ```bash
  curl -s -X POST -H "$H_AUTH" -H "$H_JSON" \
    -d "{\"entity_id\":\"$TTS_TARGET\",\"message\":\"test\"}" \
    "$API_URL/api/services/tts/speak"
  ```
  If `tts_default_target` is null in config, ask the user which `media_player.*` to use.
- **Log review** — `tail -100 $CONFIG_PATH/home-assistant.log` over SSH, or `GET /api/error_log`.
- **Config validation** — always validate before restarting after a YAML edit.

## Guidelines

- **Prefer the REST API for entity state queries and service calls** — cleaner, atomic, and doesn't require SSH.
- **For YAML edits, SSH in, edit, validate, then restart on confirmation.** Never restart HA without explicit confirmation — it interrupts running automations and integrations.
- **Back up YAML files before substantive edits.** `cp automations.yaml automations.yaml.bak.<timestamp>`.
- **TTS / announcements** — always use `tts_default_target` from config as the default. Don't assume any specific `media_player.*` exists.
- **Don't expose the long-lived access token in logs, output, or commit history.** Resolve it from `api_token_ref` only when needed.
- **Validate before restart.** `ha core check` on HAOS/Supervised, or `POST /api/config/core/check_config` over the API.
