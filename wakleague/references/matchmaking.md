# WakLeague - Matchmaking System

## Overview

Automatic matchmaking system that pairs players based on ELO, format, server, level range, and queue time. Runs every **30 seconds**.

## Matchmaking Loop

```
backgroundTimerTask()
    │
    ├── FetchQueuedPlayers()
    ├── CreateCouples(players)
    ├── For each pair:
    │   └── If score >= 0.9: CreateMatch()
    ├── Pass 1: Ranked (mode=0)
    ├── Pass 2: Chill (mode=1)
    └── CheckPendingMatches()
```

---

## Match Score Calculation

### Formula

```
note = exp(-diff²/(2*σ²)) × K × Q × lop_val × F × M
```

### Components

| Variable | Description | Range |
|----------|-------------|-------|
| `diff` | ELO difference | 0 - ∞ |
| `σ` (sigma) | ELO weight | Affected by wait time |
| `K` | Level compatibility | 0 or 1 |
| `Q` | Server compatibility | 0 or 1 |
| `F` | Format compatibility | 0 or 1 |
| `M` | Mode compatibility | 0 or 1 |
| `lop_val` | Not fought recently | 0.5 - 1.0 |

### Sigma Calculation

```python
sigma = BASE_SIGMA + (wait_time * SIGMA_PER_SECOND)
# BASE_SIGMA = 200
# SIGMA_PER_SECOND = 2
# Longer wait = higher sigma = more ELO difference allowed
```

### Compatibility Functions

```python
def checkLevel(player1, player2):
    """1 if level ranges overlap, else 0"""
    return 1 if ranges_overlap else 0

def getLopValue(player1, player2):
    """0.5-1.0 based on time since last match"""
    if < 1 min: return 0.5
    if < 5 min: return 0.7
    if < 10 min: return 0.85
    return 1.0
```

---

## Match Criteria

```python
MIN_MARK_FOR_MATCH = 0.9    # Minimum score
MAX_ELO_DIFF = 1500         # Hard cap
MAX_WAIT_TIME = 9           # Base seconds
WAIT_TIME_OFFSET = 180      # Added to max wait
```

### Match Conditions

```python
can_match = (
    score >= MIN_MARK_FOR_MATCH and
    abs(elo_diff) <= MAX_ELO_DIFF and
    same_format and
    same_mode and
    compatible_levels and
    not_recently_played
)
```

---

## Queue Entry

```python
{
    "id": discord_id,
    "lvl": [min_level, max_level],
    "queue": server_id,      # 1-4
    "format": format_id,    # 1-6
    "mode": mode_id,        # 0=Ranked, 1=Chill
    "mate1-5": teammate_ids,
    "time": timestamp
}
```

---

## Match Creation

### Channel Creation

```python
async def createMatchChannel(match):
    category = getChannel(CATEGORY_MATCH_TEXTCHANNEL)
    name = f"match-{match.id}-{format}v{format}"
    
    # Permissions for players + MOD
    overwrites = {...}
    
    channel = await category.create_text_channel(name, overwrites=overwrites)
    match.discordchannel = channel.id
    
    await sendMatchStartMessage(channel, match)
```

---

## Match States

```
QUEUE → MATCH_FOUND → WAITING_CONFIRM → READY → IN_PROGRESS → COMPLETED
                       ↓
                   TIMEOUT (cancelled)
```

| State | Description |
|-------|-------------|
| `QUEUE` | User in queue table |
| `MATCH_FOUND` | Match created, DMs sent |
| `WAITING_CONFIRM` | Players confirming class/pseudo |
| `READY` | All players confirmed |
| `IN_PROGRESS` | Match active |
| `COMPLETED` | Result submitted |
| `TIMEOUT` | Confirmation timeout (300s) |

---

## Player Confirmation Flow

### 1. Class/Pseudo Selection

DM with `ClassAndPseudoView`:
- Select class from dropdown
- Confirm button
- Mark player as confirmed

### 2. Result Reporting

`MatchResultView` appears when ready:
- Victory button → Team 1 wins
- Defeat button → Team 2 wins  
- Dispute button → Referee intervention

### 3. Confirmation

Other team confirms:
- Both agree → `concludeMatch()`
- Disputed → Notify referees

---

## Timeout Handling

```python
MATCH_CONFIRM_TIMEOUT = 300   # 5 minutes
MATCH_RESULT_TIMEOUT = 600    # 10 minutes
AFK_CHECK_INTERVAL = 60        # 1 minute
```

| Timeout | Action |
|---------|--------|
| Confirm | Cancel match, refund queue |
| Result | Cancel match, penalize reporter |
| AFK check | Reminder, then removal |

---

## Team Queue

For formats 2v2-5v5, players queue with pre-made teams:

```python
# Group leader creates party
/group create

# Add teammates
/group add @Player2 @Player3

# Queue together
QueueView → Select format with group
```

---

## Configuration

```json
{
    "MATCH_CONFIRM_WAITTIME": 300,
    "MATCH_RESULT_WAITTIME": 600,
    "MIN_MARK_FOR_MATCH": 0.9,
    "MAX_ELO_DIFF": 1500,
    "MAX_WAIT_TIME": 9,
    "WAIT_TIME_OFFSET": 180,
    "BASE_SIGMA": 200,
    "SIGMA_PER_SECOND": 2
}
```
