# WakLeague - Discord PvP Matchmaking Bot

---
name: "WakLeague"
description: "Discord PvP Matchmaking Bot for the game Wakfu. Manages registration, ELO, matchmaking, clans, tournaments and more."
author: "utojiko"
tags:
    - discord
    - matchmaking
    - wakfu
---

## Overview

**WakLeague** is a Discord bot for competitive PvP matchmaking on the game **Wakfu**. It manages player registration, automatic matchmaking with ELO ranking, tournaments, clans, battle pass, achievements, pets, and chests.

- **Language**: Python 3.x
- **Discord Library**: discord.py (async)
- **Database**: SQLite via APSW
- **Architecture**: Module-based with commands, views, tasks, embeds

## Project Structure

```
wakleague/
├── bot.py                    # Main entry point - Bot class
├── config.py                 # Configuration loader
├── dbhand.py                 # Database handler (APSW wrapper)
├── initdb.py                 # Database schema initialization
├── helpers.py                # Shared utilities
├── utils.py                  # General utilities
├── elo_utils.py              # ELO calculation
├── user.py                   # WakMMUser class (player model)
├── match.py                  # WakMMMatch class (match model)
├── clan.py                  # WakMMClan class (clan model)
├── group.py                  # Group system (team queue)
├── draft.py                  # Draft/pick-ban system
├── event.py                  # Event system
├── lang_channels.py          # Language channel utilities
├── commands/                 # Slash commands
│   ├── admin.py
│   ├── player.py
│   ├── draft.py
│   └── addevent.py
├── views/                    # Discord UI components
│   ├── queue.py
│   ├── register.py
│   ├── match_found.py
│   ├── match_result.py
│   ├── profile.py
│   ├── clan_*.py
│   ├── chest.py
│   ├── pet.py
│   ├── achievements.py
│   ├── standings.py
│   └── ...
├── embeds/                   # Embed builders
│   ├── generic.py
│   ├── match_news.py
│   ├── queue_embed.py
│   └── misc.py
├── tasks/                    # Background tasks
│   ├── matchmaking.py
│   ├── events.py
│   ├── weekly.py
│   └── daily.py
├── translation/              # Language files
│   ├── french.json
│   ├── english.json
│   ├── spanish.json
│   └── portuguese.json
├── battlepass/               # Battle pass system
├── achievements/             # Achievement system
├── chest/                    # Chest rewards system
├── pet/                      # Pet/familiar system
├── config.json               # Config
├── env_alpha.json            # Alpha settings
├── env_offi.json            # Production
├── event.json               # Events data
├── titles.json              # Titles
├── wakdb.db                 # SQLite database
└── assets/                   # Images and assets
```

## Key Classes

| Class | File | Purpose |
|-------|------|---------|
| `Bot` | `bot.py` | Main Discord client, registers commands/tasks |
| `WakMMUser` | `user.py` | Player data, ELO, queue management |
| `WakMMMatch` | `match.py` | Match lifecycle, result reporting |
| `WakMMClan` | `clan.py` | Clan management, invites |
| `Draft` | `draft.py` | Pick-ban system for 6v6 |
| `DBHandling` | `dbhand.py` | Thread-safe database access |

## Database Models

### WakMMUser (`user.py`)

Player data model with per-format, per-server ELO tracking.

**Key Properties:**
- `discordid` - Discord user ID (primary key)
- `pseudo` - Wakfu character name
- `elo{1-6}{1-4}` - ELO per format (1-6) and server (1-4)
- `winstreak{1-6}{1-4}` - Win streaks
- `credits` - Currency (WakLingots)
- `chestkeydraw`, `chestkeymoney` - Chest keys
- `titles` - JSON array of owned titles
- `clan` - Clan ID
- `language` - 0=FR, 1=EN, 2=ES, 3=PT

**Key Methods:**
```python
def getELO(format, server) -> int
def updateELO(format, server, delta)
def addToQueue(format, server, mode, lvl_range, mates=[])
def removeFromQueue()
def isInQueue() -> bool
def addCredits(amount)
def addTitle(title_id)
```

### WakMMMatch (`match.py`)

Match data model for tracking match lifecycle.

**Key Properties:**
- `usr1id`, `usr2id` - Team leaders
- `usr1m1id...usr1m5id` - Team 1 members
- `usr2m1id...usr2m5id` - Team 2 members
- `format` - Format (1-6)
- `queue` - Server (1-4)
- `mode` - 0=Ranked, 1=Chill
- `result` - 1=team1 wins, 2=team2 wins, -1=cancelled
- `discordchannel` - Match channel ID

**Key Methods:**
```python
def createNewMatch(players, format, server, mode) -> WakMMMatch
def getTeam1() -> list[WakMMUser]
def getTeam2() -> list[WakMMUser]
def addResult(result)
def concludeMatch()
```

### WakMMClan (`clan.py`)

Clan management.

**Key Methods:**
```python
def createClan(owner_id, name) -> WakMMClan
def addMember(user_id)
def removeMember(user_id)
def invitePlayer(user_id)
def transferOwnership(new_owner_id)
```

### Queue Table (Temporary)

Stores active queue entries.
```sql
-- Fields: id, lvl (JSON), queue, format, mode, mate1-5, time
```

### Draft (`draft.py`)

Pick-ban system for 6v6 tournaments.

**Key Methods:**
```python
def create(captain1, captain2, players1, players2)
def addBan(player_id, character)
def addPick(player_id, character)
def nextTurn()
```

## Matchmaking System

### Flow (`tasks/matchmaking.py`)

Background task runs every **30 seconds**.

```
backgroundTimerTask()
    ├── 1. Fetch queued players
    ├── 2. Create all possible pairs
    ├── 3. Calculate match score for each pair
    ├── 4. Match pairs with score >= 0.9
    ├── 5. Two passes: Ranked (mode=0), Chill (mode=1)
    └── 6. Check pending matches for timeouts
```

### Match Score Formula

```
note = exp(-diff²/(2*σ²)) × K × Q × lop_val × F × M
```

| Variable | Description |
|----------|-------------|
| `diff` | ELO difference |
| `σ` (sigma) | ELO weight (increases with wait time) |
| `K` | Level compatibility (0 or 1) |
| `Q` | Server compatibility (0 or 1) |
| `F` | Format compatibility (0 or 1) |
| `M` | Mode compatibility (0 or 1) |
| `lop_val` | Not fought recently (0.5-1.0) |

### Match Criteria

```python
MIN_MARK_FOR_MATCH = 0.9
MAX_ELO_DIFF = 1500
MAX_WAIT_TIME = 9 + 180 seconds
```

### Match States

```
QUEUE → MATCH_FOUND → WAITING_CONFIRM → READY → IN_PROGRESS → COMPLETED
                       ↓
                   TIMEOUT (cancelled)
```

## Commands

### Admin Commands (`commands/admin.py`)

| Command | Description | Required Role |
|---------|-------------|--------------|
| `/register` | Post registration message | MODO |
| `/clanmessage` | Post clan management message | MODO |
| `/search` | Post matchmaking search interface | MODO |
| `/setup_language_channels` | Create language categories | ADMIN |
| `/say` | Bot speaks a message | ADMIN |
| `/sync_nicknames` | Sync Discord nicknames | ADMIN |
| `/community` | Post community role selection | MODO |
| `/secondrole` | Post secondary role selection | MODO |

### Player Commands (`commands/player.py`)

| Command | Description |
|---------|-------------|
| `/profil` | View player profile (context menu) |
| `/stats` | View detailed statistics |
| `/clan` | Create/view/manage clan |
| `/group` | Manage party for team queue |
| `/pet` | View/manage pet |
| `/shop` | Open the shop |
| `/chest` | Open chests with keys |
| `/event` | View current event |
| `/battlepass` | View battle pass progress |
| `/exploits` | View achievements |
| `/help` | Get help |

### Draft Commands (`commands/draft.py`)

| Command | Description |
|---------|-------------|
| `/draft` | Start training draft |
| `/draft2` | Start tournament draft |
| `/info` | Get draft info |
| `/add` | Add spectator |
| `/setfooter` | Set draft footer |
| `/settitle` | Set draft title |

## Views (Discord UI)

### Pattern

```python
class MyView(ui.View):
    def __init__(self, ctx_data: dict = None):
        super().__init__(timeout=300)
        self.ctx_data = ctx_data or {}
    
    @ui.button(label="Action", style=discord.ButtonStyle.green)
    async def action_button(self, interaction: discord.Interaction, button: ui.Button):
        await interaction.response.edit_message(content="Updated!")
    
    @ui.select(cls=ui.Select, placeholder="Choose...")
    async def select_callback(self, interaction: discord.Interaction, select: ui.Select):
        await interaction.response.edit_message(content=f"Selected: {select.values[0]}")
```

### Key Views

| View | File | Purpose |
|------|------|---------|
| `QueueView` | `views/queue.py` | Join/Rematch/Cancel queue |
| `EnterQueueView` | `views/queue.py` | Select format/level/server |
| `RegisterView` | `views/register.py` | Character registration |
| `ClassAndPseudoView` | `views/match_found.py` | Select class for match |
| `MatchResultView` | `views/match_result.py` | Report Victory/Defeat/Dispute |
| `ProfileView` | `views/profile.py` | Profile navigation |
| `ClanView` | `views/clan_*.py` | Clan management |
| `ChestView` | `views/chest.py` | Open chests |
| `PetView` | `views/pet.py` | Pet management |

## ELO System (`elo_utils.py`)

### Parameters

```python
ELO_BASE = 125                    # Base K-factor
MAX_K_FACTOR = 100                # Maximum K-factor
MIN_K_FACTOR = 40                 # Minimum K-factor
ELO_STAGE_K_FACTOR = 2000         # ELO threshold for reduced K

WINSTREAK_STAGE1 = 3              # 1.3x bonus at 3 wins
WINSTREAK_STAGE2 = 7              # 1.6x bonus at 7 wins
BONUS_FIRST_WIN = 2               # First win of day multiplier
```

### K-Factor Calculation

K-factor scales with ELO (high ELO = smaller K) and win streaks (longer streaks = bigger K).

```python
def getKFactor(elo: int, win_streak: int) -> float:
    k = ELO_BASE
    if elo >= ELO_STAGE_K_FACTOR:
        reduction = min(elo - ELO_STAGE_K_FACTOR, 3000) / 50
        k = max(MAX_K_FACTOR - reduction, MIN_K_FACTOR)
    
    if win_streak >= WINSTREAK_STAGE2:
        k *= WINSTREAK_MULTIPLIER2
    elif win_streak >= WINSTREAK_STAGE1:
        k *= WINSTREAK_MULTIPLIER1
    
    return k
```

### ELO Change

```python
def calculateEloChange(winner_elo, loser_elo, winner_streak, loser_streak, is_first_win=False):
    expected = 1 / (1 + 10 ** ((loser_elo - winner_elo) / 400))
    delta = getKFactor(winner_elo, winner_streak) * (1 - expected)
    
    if is_first_win:
        delta *= BONUS_FIRST_WIN
    
    return round(delta), round(-delta)
```

### Rank Tiers

| Rank | ELO Range |
|------|-----------|
| Bronze | 0-1399 |
| Silver | 1400-1899 |
| Gold | 1900-2399 |
| Platinum | 2400-2899 |
| Diamond | 2900-3399 |
| Champion | 3400-3499 |
| Grandmaster | 3500+ |

## Background Tasks (`tasks/`)

| Task | Interval | Description |
|------|----------|-------------|
| `matchmaking.py` | 30s | Process queue, create matches |
| `events.py` | 61s | Monitor events, send notifications |
| `weekly.py` | Weekly | Season maintenance, rewards |
| `daily.py` | Daily | Update standings, reset stats |

### Matchmaking Task

```python
@tasks.loop(seconds=30)
async def backgroundTimerTask():
    await processMatches(mode=0)  # Ranked
    await processMatches(mode=1)  # Chill
    await checkPendingMatches()   # Timeouts
```

## Configuration

### Config Files

- `env_alpha.json` - Development settings
- `env_offi.json` - Production settings
- `event.json` - Scheduled events
- `titles.json` - Player titles
- `translation/*.json` - Language strings

### Key Config Sections

```json
{
    "DISCORD_TOKEN": "bot-token",
    "SERVERS": ["Pandora", "Rubilax", "Ogrest", "Beta"],
    "SEASON": {"number": 5, "name": "Season 5"},
    "MATCH_CONFIRM_WAITTIME": 300,
    "MIN_MARK_FOR_MATCH": 0.9,
    "ELO_BASE": 125
}
```

### Load Config

```python
from config import Config
Config.load("alpha")  # or "offi"
token = Config.get("DISCORD_TOKEN")
```

## Server/Format System

### Servers
1. Pandora
2. Rubilax
3. Ogrest
4. Beta

### Formats
1. 1v1
2. 2v2
3. 3v3
4. 4v4
5. 5v5
6. 6v6 (Draft mode)

### Modes
- 0: Ranked
- 1: Chill/Draft

## Language Support

| Code | Language | File |
|------|----------|------|
| 0 | French | `french.json` |
| 1 | English | `english.json` |
| 2 | Spanish | `spanish.json` |
| 3 | Portuguese | `portuguese.json` |

## Adding New Features

### Add a Command

1. Create `commands/newname.py`:
```python
@app_commands.describe(param="Description")
async def newname(interaction: discord.Interaction, param: str):
    await interaction.response.send_message(f"Result: {param}")

def setup(tree, guild_ids=None):
    tree.add_command(app_commands.Command(name="newname", description="...", callback=newname), guilds=guild_ids)
```

2. Register in `bot.py`:
```python
from commands import newname
newname.setup(bot.tree)
```

### Add a View

1. Create `views/newname.py`:
```python
class NewView(ui.View):
    @ui.button(label="Click", style=discord.ButtonStyle.green)
    async def click(self, interaction, button):
        await interaction.response.edit_message(content="Clicked!")
```

2. Use in command:
```python
await interaction.response.send_message("Test", view=NewView())
```

### Add a Background Task

1. Create `tasks/newname.py`:
```python
@tasks.loop(minutes=15)
async def newTask():
    await do_work()

@newTask.before_loop
async def before():
    await bot.wait_until_ready()
```

2. Register in `bot.py`:
```python
from tasks import newname
newname.newTask.start()
```

## Common Patterns

### Permission Check
```python
def has_role(member, role_id):
    return any(r.id == role_id for r in member.roles)

if not has_role(interaction.user, CONFIG["ROLE_MOD"]):
    await interaction.response.send_message("Not authorized", ephemeral=True)
    return
```

### Database Query
```python
db = DBHandling()
user = db.fetchone("SELECT * FROM users WHERE discordid = ?", (user_id,))
users = db.fetchall("SELECT * FROM users WHERE register = ?", (1,))
db.execute("UPDATE users SET credits = ? WHERE discordid = ?", (credits, user_id))
```

### Interaction Response
```python
# Edit original
await interaction.response.edit_message(content="Updated", view=view)

# Send new (ephemeral)
await interaction.response.send_message("Private", ephemeral=True)

# Send modal
await interaction.response.send_modal(MyModal())

# Defer for long ops
await interaction.response.defer()
await edit_original_response(content="Done")
```

## Run Commands

```bash
pip install discord.py apsw numpy
python initdb.py  # Create database
python bot.py     # Start bot
```
