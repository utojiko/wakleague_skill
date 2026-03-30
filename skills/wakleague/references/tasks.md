# WakLeague - Background Tasks

## Overview

Background tasks run periodically for matchmaking, events, and maintenance.

## Task Registration

In `bot.py` `setup_hook()`:

```python
async def setup_hook():
    await bot.tree.sync()
    
    tasks.matchmaking.backgroundTimerTask.start()
    tasks.events.eventTimerTask.start()
    tasks.weekly.weeklyTask.start()
    tasks.daily.dailyTask.start()
```

---

## Matchmaking Task (`tasks/matchmaking.py`)

Core matchmaking loop.

### Configuration

```python
@tasks.loop(seconds=30)
async def backgroundTimerTask():
    pass
```

### Process Flow

```python
@tasks.loop(seconds=30)
async def backgroundTimerTask():
    try:
        await processMatches(mode=0)  # Ranked
        await processMatches(mode=1)  # Chill
        await checkPendingMatches()
    except Exception as e:
        log_error(f"Matchmaking error: {e}")

async def processMatches(mode: int):
    queued_players = getQueuedPlayers(mode)
    if len(queued_players) < 2:
        return
    
    # Generate all pairs
    pairs = generatePairs(queued_players)
    
    # Score each pair
    scored_pairs = []
    for pair in pairs:
        score = calculateMatchScore(pair)
        if score >= MIN_MARK_FOR_MATCH:  # 0.9
            scored_pairs.append((pair, score))
    
    # Sort and match
    scored_pairs.sort(key=lambda x: x[1], reverse=True)
    
    matched = set()
    for pair, score in scored_pairs:
        p1, p2 = pair
        if p1.id in matched or p2.id in matched:
            continue
        
        await createMatch(p1, p2)
        matched.add(p1.id)
        matched.add(p2.id)
```

### Match Creation

```python
async def createMatch(p1, p2):
    p1.removeFromQueue()
    p2.removeFromQueue()
    
    match = WakMMMatch.createNewMatch(
        players=[p1, p2],
        format=p1.format,
        server=p1.server,
        mode=p1.mode
    )
    
    channel = await createMatchChannel(match)
    
    for player in [p1, p2]:
        await sendMatchFoundDM(player, match, channel)
```

### Channel Creation

```python
async def createMatchChannel(match) -> discord.TextChannel:
    guild = bot.get_guild(CONFIG["SERVER_ID"])
    category = discord.utils.get(guild.categories, id=CONFIG["CATEGORY_MATCH_TEXTCHANNEL"])
    
    overwrites = {guild.default_role: discord.PermissionOverwrite(read_messages=False)}
    
    for player in match.getTeam1() + match.getTeam2():
        member = guild.get_member(int(player.discordid))
        overwrites[member] = discord.PermissionOverwrite(read_messages=True, send_messages=True)
    
    channel = await category.create_text_channel(
        name=f"match-{match.id}-{match.format}v{match.format}",
        overwrites=overwrites
    )
    return channel
```

### Pending Match Checks

```python
async def checkPendingMatches():
    for match in WakMMMatch.getPendingMatches():
        if match.isExpired(CONFIRM_TIMEOUT):
            await cancelMatch(match, reason="timeout")
        elif match.isExpired(RESULT_TIMEOUT):
            await cancelMatch(match, reason="no_result")
        elif match.needsAFKReminder():
            await sendAFKReminder(match)
```

---

## Event Task (`tasks/events.py`)

Event notifications.

### Configuration

```python
@tasks.loop(seconds=61)
async def eventTimerTask():
    pass
```

### Process Flow

```python
@tasks.loop(seconds=61)
async def eventTimerTask():
    await checkStartingEvents()
    await checkEndingEvents()
    await sendEventReminders()

async def checkStartingEvents():
    events = loadEvents()
    now = time.time()
    
    for event in events:
        start = event.get("start", 0)
        if 0 < (start - now) <= 60 and not event.get("notified_start"):
            await announceEventStart(event)
            event["notified_start"] = True
            saveEvents(events)

async def checkEndingEvents():
    events = loadEvents()
    for event in events:
        if event.get("end", 0) < time.time() and not event.get("concluded"):
            await concludeEvent(event)
            event["concluded"] = True
            saveEvents(events)
```

### Event Announcement

```python
async def announceEventStart(event: dict):
    for channel in EVENT_CHANNELS:
        embed = discord.Embed(
            title=f"🎮 {event['name']}",
            color=discord.Color.gold()
        )
        embed.add_field(name="ELO Multiplier", value=f"x{event.get('elo_multiplier', 1.0)}")
        
        if event.get("class_allowed"):
            classes = ", ".join(getClassNames(event["class_allowed"]))
            embed.add_field(name="Classes", value=classes)
        
        await channel.send("@everyone Event starting!", embed=embed)
```

---

## Weekly Task (`tasks/weekly.py`)

Weekly maintenance.

### Configuration

```python
@tasks.loop(hours=168)  # Every 7 days
async def weeklyTask():
    pass

@weeklyTask.before_loop
async def before_weekly():
    await bot.wait_until_ready()
```

### Process Flow

```python
@tasks.loop(hours=168)
async def weeklyTask():
    await resetWeeklyStats()
    await updateWeeklyLeaderboard()
    await awardWeeklyRewards()
    await generateWeeklyReport()

async def resetWeeklyStats():
    db.execute("UPDATE users SET weekly_wins = 0")
    db.execute("UPDATE users SET weekly_elo_gain = 0")
```

### Weekly Rewards

```python
async def awardWeeklyRewards():
    top_players = getWeeklyTopPlayers(limit=10)
    
    rewards = [
        {"place": 1, "credits": 5000, "title": "weekly_1"},
        {"place": 2, "credits": 3000, "title": "weekly_2"},
        {"place": 3, "credits": 2000, "title": "weekly_3"},
    ]
    
    for i, player in enumerate(top_players[:3]):
        player.addCredits(rewards[i]["credits"])
        player.addTitle(rewards[i]["title"])
```

---

## Daily Task (`tasks/daily.py`)

Daily maintenance.

### Configuration

```python
@tasks.loop(hours=24)
async def dailyTask():
    pass

@dailyTask.before_loop
async def before_daily():
    await bot.wait_until_ready()
```

### Process Flow

```python
@tasks.loop(hours=24)
async def dailyTask():
    await updateAllStandings()
    await resetDailyStats()
    await applyEloDecay()
    await generateDailyReport()

async def updateAllStandings():
    for format in range(1, 7):
        for server in range(1, 5):
            await updateStandings(format, server)

async def resetDailyStats():
    db.execute("UPDATE users SET daily_wins = 0")
    db.execute("UPDATE users SET daily_first_win = 0")
```

### Standings Update

```python
async def updateStandings(format: int, server: int):
    channel = getChannelForStandings(format, server)
    
    rankings = db.fetchall("""
        SELECT discordid, elo{format}_{server} as elo
        FROM users WHERE register = 1
        ORDER BY elo DESC LIMIT 25
    """)
    
    embed = buildStandingsEmbed(rankings, format, server)
    msg = await channel.fetch_message(channel.last_message_id)
    await msg.edit(embed=embed)
```

---

## Task Utilities

### Error Handling

```python
async def safe_task(coro):
    try:
        await coro()
    except Exception as e:
        log_error(f"Task error: {traceback.format_exc()}")
```

### Logging

```python
def log_task_start(task_name):
    log(f"[TASK] Starting: {task_name}")

def log_task_end(task_name, duration):
    log(f"[TASK] Completed: {task_name} in {duration:.2f}s")
```

---

## Configuration

```python
TASK_CONFIG = {
    "matchmaking": {"interval": 30, "enabled": True},
    "events": {"interval": 61, "enabled": True},
    "weekly": {"interval_hours": 168, "enabled": True},
    "daily": {"interval_hours": 24, "enabled": True}
}
```

---

## Manual Execution

```python
@bot.tree.command(name="runtask")
async def runtask(interaction, task: str):
    if task == "matchmaking":
        await processMatches(0)
        await processMatches(1)
    elif task == "standings":
        await updateAllStandings()
```
