---
name: onboard
description: First-run onboarding for the home-assistant-mgmt plugin. Interview the user about their Home Assistant instance — host, install type (HAOS / Container / Core / Supervised), SSH access, REST API URL, long-lived token reference, and (optionally) a default TTS target — and persist the result to `$CLAUDE_USER_DATA/home-assistant-mgmt/config.json`. Run this before any other skill in this plugin, or whenever the connection details change. Triggers on phrases like "set up home assistant", "onboard ha", "configure home assistant plugin".
---

# home-assistant-mgmt: onboard

Establish the persistent connection profile for a Home Assistant instance. The same plugin install can serve multiple HA environments — only the values in `config.json` are environment-specific. No host details should ever be hard-coded into other skills in this plugin.

## Workspace resolution

Resolve the plugin's data directory as `$CLAUDE_USER_DATA/home-assistant-mgmt/` if `CLAUDE_USER_DATA` is set; otherwise `$XDG_DATA_HOME/claude-plugins/home-assistant-mgmt/` if `XDG_DATA_HOME` is set; otherwise `~/.local/share/claude-plugins/home-assistant-mgmt/`. Create the directory if it doesn't exist. See the canonical convention in the `claude-rudder:plugin-data-storage` skill.

Shell form:

```bash
PLUGIN_DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/home-assistant-mgmt"
mkdir -p "$PLUGIN_DATA_DIR"
```

The config file is `$PLUGIN_DATA_DIR/config.json`.

## When to use

- User says "set up home assistant", "onboard ha", "configure home assistant plugin".
- Any other home-assistant-mgmt skill finds no `config.json` at the resolved path — offer to run onboarding first.
- The user changes hosts, install types, or credentials.

## Procedure

1. **Load** existing config from `$PLUGIN_DATA_DIR/config.json` if present. Show the current values back. Offer **Update** vs **Replace** vs **Cancel**.

2. **Interview** the user. Ask each question; default sensibly when the user just hits enter:

   | Field | Prompt | Notes |
   |---|---|---|
   | `host` | "What's the LAN address of your Home Assistant host?" | No default — user must answer. Hostname or IP. |
   | `install_type` | "Install type? (`haos` / `container` / `core` / `supervised`)" | Affects available CLIs (`ha core check` works on HAOS/Supervised; not on Core/Container). |
   | `ssh_enabled` | "Is SSH access available? (y/n)" | If yes, ask for the next three. |
   | `ssh_user` | "SSH username?" | Default depends on install type: `root` (HAOS via Add-on), or whatever the user set. |
   | `ssh_port` | "SSH port?" | Default: `22` (or `22222` for the HAOS SSH add-on). |
   | `ssh_key_path` | "Path to the SSH private key (or leave blank for default agent)?" | Optional. Expand `~`. |
   | `api_url` | "REST API base URL?" | Default: `http://<host>:8123`. Use HTTPS if HA is behind a reverse proxy. |
   | `api_token_ref` | "Where is the long-lived access token stored? (1Password ref / env var name / file path)" | Reference only — never the token itself. Create one at `<api_url>/profile/security`. |
   | `tts_default_target` | "Default TTS / media_player target for testing? (e.g. `media_player.living_room_speaker`)" | Optional. Used by skills that test announcements. |
   | `config_path` | "Path to the HA config dir (when SSH'd in)?" | Default: `/config` (HAOS/Supervised/Container) or `~/.homeassistant` (Core). |
   | `notes` | "Any free-text notes about this HA install?" | Optional. |

3. **Test connectivity** (best-effort, don't block on failure):

   - **API reachability** (always, if `api_token_ref` resolved):
     ```bash
     curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/api/" | head -c 200
     ```
     Expect `{"message":"API running."}`.
   - **SSH** (if `ssh_enabled`):
     ```bash
     ssh -o ConnectTimeout=5 -o BatchMode=yes -p "$SSH_PORT" "$SSH_USER@$HOST" \
       "ha core info 2>/dev/null || cat /config/.HA_VERSION 2>/dev/null || echo no-version"
     ```

   Surface failures but still write the config.

4. **Write** `$PLUGIN_DATA_DIR/config.json` with this schema (pretty-printed, 2-space indent):

   ```json
   {
     "schema_version": 1,
     "host": "192.168.1.20",
     "install_type": "haos",
     "ssh": {
       "enabled": false,
       "user": "root",
       "port": 22,
       "key_path": null
     },
     "api_url": "http://192.168.1.20:8123",
     "api_token_ref": null,
     "tts_default_target": null,
     "config_path": "/config",
     "notes": null,
     "updated_at": "<ISO-8601 timestamp>"
   }
   ```

5. **Summarise**: print every field and its final value. Show the path to `config.json`. List the sibling skills the user can now invoke (`home-assistant-ops`).

## Re-running

If `config.json` already exists, never silently overwrite. Always confirm. If the user picks **Replace**, back up the existing file with `.bak` suffix first.

## Hard rules

- **Never store the long-lived access token in `config.json`.** Only store a reference. The seed skill resolves it at runtime.
- **Never hard-code IPs, hostnames, usernames, or entity IDs into other skills.** Every operational skill must read from `config.json`.
- **`tts_default_target` is environment-specific.** Don't assume any particular `media_player.*` exists in any other user's HA — always read it from config.
