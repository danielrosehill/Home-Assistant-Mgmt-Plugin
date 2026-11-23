# Home Assistant Management Plugin

A Claude Code plugin for managing a [Home Assistant](https://www.home-assistant.io/) instance. REST API + SSH-based ops for automations, entities, service calls, TTS, and logs.

Per-host details (IP, install type, SSH access, API URL, long-lived token reference, default TTS target) are stored **outside the plugin** at `$CLAUDE_USER_DATA/home-assistant-mgmt/config.json`, so the same install works against any number of Home Assistant environments and survives plugin updates.

## Skills

- `onboard` — interactive first-run setup. Captures host, install type (HAOS / Container / Core / Supervised), optional SSH access, REST API URL, long-lived token reference, default TTS target, and config dir. Writes `config.json`.
- `home-assistant-ops` — automation/entity authoring, service calls, TTS testing, log review, and config validation via the HA REST API and (optionally) SSH. Reads from `config.json`.
- `group-automations` — group automations with a label and turn them on or off as a batch.
- `pause-automations` — pause automations between dates with AutoSnooze; they resume by themselves.

## Requirements

- A long-lived access token from Home Assistant: **Profile → Security → Long-lived access tokens**
- (Optional) SSH access — typically via the HAOS *SSH & Web Terminal* add-on, a Container/Core install, or your own SSH server

## Installation

```
claude plugins install home-assistant-mgmt@danielrosehill
```

## Quick start

1. Install the plugin.
2. Create a long-lived access token in HA and store it somewhere referenceable (1Password, an env var, or a file).
3. Run the `onboard` skill — Claude will interview you for the connection details and write them to `$CLAUDE_USER_DATA/home-assistant-mgmt/config.json`.
4. Ask Claude things like *"check home assistant"*, *"call light.turn_on for the kitchen"*, or *"test TTS"* — it'll read the config, resolve the token, and connect.

## Storage convention

This plugin follows the [`rudder:plugin-data-storage`](https://github.com/danielrosehill/Claude-Rudder) convention:

- **Plugin code** lives at the install path (read-only, replaced on update).
- **User config** lives at `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/home-assistant-mgmt/config.json`.
- **Secrets** (long-lived access token) are never stored in the config — only a *reference* to where they live (1Password item, env var, file path). Skills resolve the reference at runtime.

## License

MIT
