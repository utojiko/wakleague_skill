# WakLeague - Commands

## Registration

### Admin Commands (`commands/admin.py`)

| Command | Role | Description |
|---------|------|-------------|
| `/register` | MODO | Post registration message |
| `/clanmessage` | MODO | Post clan management message |
| `/search` | MODO | Post matchmaking interface |
| `/setup_language_channels` | ADMIN | Create language categories |
| `/say` | ADMIN | Bot speaks message |
| `/sync_nicknames` | ADMIN | Sync Discord nicknames |
| `/community` | MODO | Community role selection |
| `/secondrole` | MODO | Secondary role selection |

### `/register`

```python
@app_commands.describe(channel="Target channel")
async def register(interaction, channel: discord.TextChannel):
    # Creates RegisterView with language selection
    # Posts embed with FR/EN/ES/PT buttons
```

### `/search`

```python
async def search(interaction, channel: discord.TextChannel):
    # Posts QueueView
    # Posts standings alongside
```

### `/setup_language_channels`

```python
async def setup_language_channels(interaction):
    # Creates categories: FR, EN, ES, PT
    # Creates channels in each:
    # - 📝-ɪɴꜱᴄʀɪᴘᴛɪᴏɴ
    # - 📰-ʀᴜʟᴇꜱ
    # - ⚔️-ᴍᴀᴛᴄʜᴍᴀᴋɪɴɢ
    # - 🏅-ᴄʟᴀꜱꜱᴇᴍᴇɴᴛ
    # - 🗝️-ᴄᴏꜰꜰʀᴇꜱ
    # - 📅-ᴇᴠᴇɴᴛꜱ
```

---

## Player Commands (`commands/player.py`)

### `/profil`

```python
@app_commands.describe(user="Player to view")
async def profil(interaction, user: discord.Member = None):
    # Shows profile with stats
    # Context menu: right-click user → Apps → Profile
```

**Embed Fields:**
```
Pseudo: Player#1234
Level: 245 | Class: Eliotrope
━━━━━━━━━━━━━━━━━━
Titles: 5 unlocked
Wins: 142 | Win Rate: 59.2%
━━━━━━━━━━━━━━━━━━
1v1 ELO: 1842 🏅
2v2 ELO: 1756 🏅
━━━━━━━━━━━━━━━━━━
Rank: #23 Global
```

### `/stats`

```python
@app_commands.describe(user="Player", format="1-6", server="1-4")
async def stats(interaction, user=None, format: int = 1, server: int = 1):
    # Detailed statistics
    # ELO history, win streaks, best performances
```

### `/clan`

```python
@app_commands.describe(action="create|info|invite|kick|leave", name="Clan name")
async def clan(interaction, action: str, user: discord.Member = None, name: str = None):
```

| Action | Description |
|--------|-------------|
| `create` | Create new clan (costs credits) |
| `info` | View clan info |
| `invite` | Invite player to clan |
| `kick` | Remove player from clan |
| `leave` | Leave current clan |
| `manage` | Clan settings (owner only) |

### `/group`

```python
@app_commands.describe(action="create|add|remove|leave|list")
async def group(interaction, action: str, user: discord.Member = None):
```

| Action | Description |
|--------|-------------|
| `create` | Create new group (you become leader) |
| `add` | Add user to group |
| `remove` | Remove user from group |
| `leave` | Leave current group |
| `list` | List current group members |

### `/pet`

```python
@app_commands.describe(action="view|feed|rename|release")
async def pet(interaction, action: str = "view", pet_name: str = None):
```

| Action | Description |
|--------|-------------|
| `view` | See pet stats |
| `feed` | Feed pet (costs credits, increases happiness) |
| `rename` | Rename pet |
| `release` | Release pet (loses it) |

### `/shop`

```python
@app_commands.describe(category="Category filter")
async def shop(interaction, category: str = None):
```

Categories: Titles, Cosmetics, Clan, Keys

### `/chest`

```python
@app_commands.describe(chest_type="draw|money")
async def chest(interaction, chest_type: str = "draw"):
```

| Type | Key | Rewards |
|------|-----|---------|
| Draw | `chestkeydraw` | Titles, cosmetics, credits |
| Money | `chestkeymoney` | Large credits, rare items |

### `/battlepass`

```python
@app_commands.describe(action="view|claim")
async def battlepass(interaction, action: str = "view"):
```

Tracks: Daily challenges, Match wins, Season goals

### `/exploits`

```python
@app_commands.describe(category="Filter")
async def exploits(interaction, category: str = None):
```

Categories: Matches, ELO, Streaks, Clans, Collection

### `/event`

```python
async def event(interaction):
    # Shows active event details
    # Duration, restrictions, rewards, maps
```

### `/help`

```python
@app_commands.describe(command="Specific command")
async def help(interaction, command: str = None):
```

---

## Draft Commands (`commands/draft.py`)

### `/draft`

```python
@app_commands.describe(opponent="Opponent captain")
async def draft(interaction, opponent: discord.Member):
```

Starts training draft with pick/ban.

### `/draft2`

```python
@app_commands.describe(captain1="Team 1", captain2="Team 2",
                       players1="Team 1 players", players2="Team 2 players")
async def draft2(interaction, captain1, captain2, players1, players2):
```

Tournament draft with full rosters.

### `/info`

```python
@app_commands.describe(draft_id="Draft ID")
async def info(interaction, draft_id: int):
```

### `/add`

```python
@app_commands.describe(channel="Draft channel", user="User to add")
async def add(interaction, channel, user: discord.Member):
```

### `/setfooter` / `/settitle`

```python
async def setfooter(interaction, footer: str, channel: discord.TextChannel):
async def settitle(interaction, title: str, channel: discord.TextChannel):
```

---

## Event Commands (`commands/addevent.py`)

### `/addevent`

```python
@app_commands.describe(name="Event name", start="Start (ISO)",
                       end="End", elo_multiplier="ELO mult (default 1.0)")
async def addevent(interaction, name: str, start: str, end: str,
                   elo_multiplier: float = 1.0, xp_multiplier: float = 1.0):
```

---

## Command Patterns

### Permission Check

```python
async def command(interaction):
    if not has_role(interaction.user, CONFIG["ROLE_MOD"]):
        await interaction.response.send_message("Not authorized", ephemeral=True)
        return
```

### Error Handling

```python
try:
    await do_something()
except ValueError as e:
    await interaction.response.send_message(f"Error: {e}", ephemeral=True)
except Exception as e:
    log_error(e)
    await interaction.response.send_message("An error occurred", ephemeral=True)
```

### Autocomplete

```python
@autocomplete(server=get_server_list)
async def command(interaction, server: str):
    ...

async def get_server_list(interaction, current: str):
    return [Choice(name=s, value=s) for s in ["Pandora", "Rubilax", "Ogrest", "Beta"]]
```
