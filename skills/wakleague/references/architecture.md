# WakLeague - Architecture

## Project Structure

```
wakleague/
├── bot.py                    # Main entry point
├── config.py                 # Configuration loader
├── dbhand.py                 # Database handler (APSW)
├── initdb.py                 # Database schema
├── helpers.py                # Shared utilities
├── utils.py                  # General utilities
├── elo_utils.py              # ELO calculations
├── user.py                   # WakMMUser
├── match.py                  # WakMMMatch
├── clan.py                   # WakMMClan
├── group.py                  # Group system
├── draft.py                  # Draft/pick-ban
├── event.py                  # Event system
├── lang_channels.py          # Language channels
├── commands/                  # Slash commands
├── views/                    # Discord UI
├── embeds/                   # Embed builders
├── tasks/                    # Background tasks
├── translation/              # Languages (FR, EN, ES, PT)
├── battlepass/               # Battle pass
├── achievements/             # Achievements
├── chest/                    # Chests
├── pet/                      # Pets
├── config.json
├── env_alpha.json            # Alpha config
├── env_offi.json            # Production config
├── event.json               # Events
├── titles.json              # Titles
└── wakdb.db                 # SQLite database
```

## Key Classes

| Class | File | Responsibility |
|-------|------|----------------|
| `Bot` | `bot.py` | Main Discord client, registers commands/tasks |
| `WakMMUser` | `user.py` | Player data, ELO, queue management |
| `WakMMMatch` | `match.py` | Match lifecycle, result reporting |
| `WakMMClan` | `clan.py` | Clan management, invites |
| `Draft` | `draft.py` | Pick-ban system for 6v6 |
| `DBHandling` | `dbhand.py` | Thread-safe database access |

## Data Flow

```
Discord Interaction
       ↓
   Command (commands/)
       ↓
   View (views/) ←→ User Input
       ↓
   Model (user.py, match.py)
       ↓
   Database (dbhand.py)
       ↓
   Task (tasks/)
       ↓
   Embed (embeds/)
       ↓
   Discord Response
```

## Bot Initialization

1. `config.py` loads environment (`alpha`/`offi`)
2. `DBHandling` connects to SQLite
3. Commands registered from `commands/`
4. Tasks started from `tasks/`
5. `on_ready`: recovers pending state
6. Bot ready for interactions

## Directory Purpose

| Directory | Content |
|-----------|---------|
| `commands/` | Slash commands (admin.py, player.py, draft.py) |
| `views/` | UI components (buttons, selects, modals) |
| `embeds/` | Embed builders (queue, match, profile) |
| `tasks/` | Background loops (matchmaking, events) |
| `translation/` | Language strings (FR, EN, ES, PT) |
| `battlepass/` | Battle pass logic |
| `achievements/` | Achievement tracking |
| `chest/` | Chest opening system |
| `pet/` | Pet management |

## Configuration Loading

```python
# config.py
Config.load("alpha")  # or "offi"
token = Config.get("DISCORD_TOKEN")
```

## Intents Required

```python
intents = discord.Intents.all()
intents.message_content = True  # For message commands
intents.members = True          # For member access
```
