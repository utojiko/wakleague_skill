# WakLeague - Development Patterns

## Project Conventions

### File Naming

- **Commands**: `commands/*.py`
- **Views**: `views/*.py`
- **Embeds**: `embeds/*.py`
- **Tasks**: `tasks/*.py`
- **Models**: `user.py`, `match.py`, `clan.py`

### Class Naming

```python
class WakMMUser:      # WakLeague MatchMaking User
class WakMMMatch:     # WakLeague MatchMaking Match
class WakMMClan:      # WakLeague MatchMaking Clan
class Bot:            # Main bot class
```

---

## Adding a Command

### 1. Create Command File

```python
# commands/newname.py
import discord
from discord import app_commands

@app_commands.describe(param="Description")
async def newname(interaction: discord.Interaction, param: str):
    """Command description for help."""
    
    # Permission check
    if not has_permission(interaction.user, "MOD"):
        await interaction.response.send_message("Not authorized", ephemeral=True)
        return
    
    # Logic
    result = do_something(param)
    
    # Response
    await interaction.response.send_message(f"Result: {result}")

def setup(tree, guild_ids=None):
    tree.add_command(
        app_commands.Command(name="newname", description="...", callback=newname),
        guilds=guild_ids
    )
```

### 2. Register in bot.py

```python
# bot.py
from commands import newname

async def setup_hook():
    newname.setup(bot.tree)
```

---

## Adding a View

### 1. Create View File

```python
# views/newname.py
import discord
from discord import ui

class NewView(ui.View):
    def __init__(self, ctx_data: dict = None):
        super().__init__(timeout=300)
        self.ctx_data = ctx_data or {}
    
    @ui.button(label="Action", style=discord.ButtonStyle.green)
    async def action_button(self, interaction: discord.Interaction, button: ui.Button):
        await interaction.response.edit_message(content="Updated!", view=None)
    
    @ui.select(cls=ui.Select, placeholder="Choose...",
               options=[discord.SelectOption(label="Opt", value="1")])
    async def select_callback(self, interaction: discord.Interaction, select: ui.Select):
        await interaction.response.edit_message(content=f"Selected: {select.values[0]}")
    
    async def on_timeout(self):
        for child in self.children:
            child.disabled = True
        await self.message.edit(view=self)
```

### 2. Use in Command

```python
@bot.tree.command()
async def useview(interaction: discord.Interaction):
    view = NewView({"user_id": interaction.user.id})
    await interaction.response.send_message("Choose:", view=view)
```

---

## Adding a Task

### 1. Create Task File

```python
# tasks/newname.py
import discord
from discord import tasks

@tasks.loop(minutes=15)
async def newTask():
    try:
        await do_periodic_work()
    except Exception as e:
        log_error(f"Task failed: {e}")

@newTask.before_loop
async def before_task():
    await bot.wait_until_ready()
    log("NewTask starting...")
```

### 2. Register in bot.py

```python
# bot.py
from tasks import newname

async def setup_hook():
    newname.newTask.start()
```

---

## Database Operations

### Using DBHandling

```python
from dbhand import DBHandling
db = DBHandling()

# Query with parameters
user = db.fetchone("SELECT * FROM users WHERE discordid = ?", (user_id,))

# Multiple rows
users = db.fetchall("SELECT * FROM users WHERE register = ?", (1,))

# Insert/Update
db.execute("UPDATE users SET credits = ? WHERE discordid = ?", (credits, user_id))
```

---

## Error Handling

### Command Errors

```python
@bot.tree.command()
async def safe_command(interaction: discord.Interaction):
    try:
        result = might_fail()
        await interaction.response.send_message(result)
    except ValueError as e:
        await interaction.response.send_message(f"Invalid: {e}", ephemeral=True)
    except PermissionError:
        await interaction.response.send_message("No permission!", ephemeral=True)
    except Exception as e:
        log_error(f"Error: {traceback.format_exc()}")
        await interaction.response.send_message("An error occurred", ephemeral=True)
```

### Task Errors

```python
@tasks.loop(minutes=5)
async def safe_task():
    try:
        await do_work()
    except Exception as e:
        log_error(f"Task failed: {e}")  # Task continues running
```

---

## Async Patterns

### Proper Await

```python
# Correct
await send_message()
result = await fetch_data()

# Wrong
send_message()  # Missing await!
```

### Gather for Parallel

```python
import asyncio

# Parallel
await asyncio.gather(*[send(m) for m in messages])

# Sequential (slower)
for m in messages:
    await send(m)
```

### Timeout

```python
import asyncio

try:
    result = await asyncio.wait_for(long_operation(), timeout=30.0)
except asyncio.TimeoutError:
    await interaction.response.send_message("Timed out!")
```

---

## Interaction Responses

```python
# Edit original
await interaction.response.edit_message(content="Updated", view=view)

# Send new message
await interaction.response.send_message("Hello")

# Ephemeral (only user sees)
await interaction.response.send_message("Private", ephemeral=True)

# Send modal
await interaction.response.send_modal(MyModal())

# Defer (for long operations)
await interaction.response.defer()
await interaction.edit_original_response(content="Done")
```

**Important:** Respond within 3 seconds. Don't respond twice.

---

## Permissions

```python
def has_role(member: discord.Member, role_id: int) -> bool:
    return any(r.id == role_id for r in member.roles)

def is_admin(member) -> bool:
    return has_role(member, CONFIG["ROLE_ADMIN"])

def is_mod(member) -> bool:
    return has_role(member, CONFIG["ROLE_MOD"]) or is_admin(member)
```

---

## Logging

```python
import logging
log = logging.getLogger("wakleague")

log.info(f"[COMMAND] {msg}")
log.error(f"[ERROR] {msg}")
log.info(f"[MATCH] {msg}")
```

---

## Translation

```python
from helpers import getLangString

user = WakMMUser(interaction.user.id)
msg = getLangString(user.language, "commands.profile.title")
```

Add to all `translation/*.json` files:
```json
{"commands": {"profile": {"title": "Player Profile"}}}
```

---

## Common Patterns

### Fetch User

```python
def getUser(discord_id: int) -> WakMMUser:
    return WakMMUser(discord_id)
```

### Create or Get

```python
def getOrCreateUser(discord_id: int) -> WakMMUser:
    user = WakMMUser(discord_id)
    if not user.exists():
        user.create()
    return user
```

### Safe Channel Send

```python
async def safe_send(channel, content=None, embed=None):
    try:
        return await channel.send(content=content, embed=embed)
    except discord.Forbidden:
        log_error(f"Cannot send to {channel.id}")
    except Exception as e:
        log_error(f"Channel send error: {e}")
```

---

## Performance Tips

### Batch Database Operations

```python
# Slow
for user in users:
    db.execute("UPDATE ...", (user.credits + 10, user.id))

# Fast
db.execute("UPDATE ... WHERE id IN (?)", ([u.id for u in users],))
```

### Cache Frequently Used Data

```python
class Cache:
    _standings_cache = {}
    _cache_time = {}
    
    @classmethod
    def get_standings(cls, format, server, ttl=60):
        key = f"{format}_{server}"
        if key in cls._standings_cache:
            if time.time() - cls._cache_time.get(key, 0) < ttl:
                return cls._standings_cache[key]
        
        data = fetch_standings(format, server)
        cls._standings_cache[key] = data
        cls._cache_time[key] = time.time()
        return data
```

---

## Run Commands

```bash
pip install discord.py apsw numpy
python initdb.py  # Create database
python bot.py     # Start bot
```
