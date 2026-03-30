# WakLeague - Configuration

## Configuration System (`config.py`)

```python
class Config:
    _instance = None
    _config = {}
    
    @classmethod
    def load(cls, environment: str = "alpha"):
        cls._config = json.load(open(f"env_{environment}.json"))
        return cls._config
    
    @classmethod
    def get(cls, key: str, default=None):
        return cls._config.get(key, default)
```

Load with: `Config.load("alpha")` or `Config.load("offi")`

---

## Environment Files

| File | Purpose |
|------|---------|
| `env_alpha.json` | Development/testing |
| `env_offi.json` | Production |

---

## Core Configuration

### Discord

```json
{
    "DISCORD_TOKEN": "bot-token",
    "APPLICATION_ID": 123456789,
    "SERVER_ID": 987654321,
    "OWNER_ID": 111222333
}
```

### Channels

```json
{
    "CHANNEL_NEWS": 123,
    "CHANNEL_QUEUE_FR": 124,
    "CHANNEL_QUEUE_EN": 125,
    "CHANNEL_STANDINGS_FR": 126,
    "CHANNEL_EVENTS_FR": 128,
    "CHANNEL_MATCH_TEXTCHANNEL": 133,
    "CATEGORY_MATCH_TEXTCHANNEL": 134,
    "CATEGORY_DRAFT": 135,
    "CATEGORY_FR": 136,
    "CATEGORY_EN": 137
}
```

### Roles

```json
{
    "ROLE_ADMIN": 200,
    "ROLE_MOD": 201,
    "ROLE_MODO": 202,
    "ROLE_PLAYER": 203,
    "ROLE_CAPTAIN": 204,
    "ROLE_REFEREE": 205
}
```

---

## Servers

```json
{
    "SERVERS": [
        {"id": 1, "name": "Pandora", "display_name": "🌍 Pandora"},
        {"id": 2, "name": "Rubilax", "display_name": "🎮 Rubilax"},
        {"id": 3, "name": "Ogrest", "display_name": "⚔️ Ogrest"},
        {"id": 4, "name": "Beta", "display_name": "🧪 Beta"}
    ]
}
```

---

## Season

```json
{
    "SEASON": {
        "number": 5,
        "name": "Season 5 - The Awakening",
        "start_date": "2024-01-15T00:00:00Z",
        "end_date": "2024-04-15T23:59:59Z",
        "is_active": true
    }
}
```

---

## Matchmaking

```json
{
    "MATCH_CONFIRM_WAITTIME": 300,
    "MATCH_RESULT_WAITTIME": 600,
    "MIN_MARK_FOR_MATCH": 0.9,
    "MAX_ELO_DIFF": 1500,
    "MAX_WAIT_TIME": 9,
    "WAIT_TIME_OFFSET": 180,
    "BASE_SIGMA": 200,
    "SIGMA_PER_SECOND": 2,
    "AFK_CHECK_INTERVAL": 60,
    "QUEUE_TIMEOUT": 3600
}
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MATCH_CONFIRM_WAITTIME` | 300s | Time to confirm class/pseudo |
| `MATCH_RESULT_WAITTIME` | 600s | Time to report result |
| `MIN_MARK_FOR_MATCH` | 0.9 | Minimum match score |
| `MAX_ELO_DIFF` | 1500 | Hard cap on ELO difference |
| `MAX_WAIT_TIME` | 9s | Base wait time |
| `WAIT_TIME_OFFSET` | 180s | Added to max wait |

---

## Levels

```json
{
    "LEVELS": [80, 110, 140, 170, 200, 230, 245],
    "LEVEL_TIERS": {
        "casual": {"min": 80, "max": 140},
        "mid": {"min": 140, "max": 200},
        "high": {"min": 200, "max": 245},
        "top": {"min": 245, "max": 250}
    }
}
```

---

## ELO

```json
{
    "ELO_BASE": 125,
    "MAX_K_FACTOR": 100,
    "MIN_K_FACTOR": 40,
    "ELO_STAGE_K_FACTOR": 2000,
    "WINSTREAK_STAGE1": 3,
    "WINSTREAK_MULTIPLIER1": 1.3,
    "WINSTREAK_STAGE2": 7,
    "WINSTREAK_MULTIPLIER2": 1.6,
    "BONUS_FIRST_WIN": 2,
    "DECAY_THRESHOLD_DAYS": 30,
    "DECAY_AMOUNT": 10,
    "INITIAL_ELO": 1000
}
```

---

## Ranks

```json
{
    "RANKS": [
        {"name": "Bronze", "min": 0, "max": 1399, "emoji": "🥉"},
        {"name": "Silver", "min": 1400, "max": 1899, "emoji": "🥈"},
        {"name": "Gold", "min": 1900, "max": 2399, "emoji": "🥇"},
        {"name": "Platinum", "min": 2400, "max": 2899, "emoji": "💎"},
        {"name": "Diamond", "min": 2900, "max": 3399, "emoji": "💠"},
        {"name": "Champion", "min": 3400, "max": 3499, "emoji": "🏆"},
        {"name": "Grandmaster", "min": 3500, "max": 9999, "emoji": "👑"}
    ]
}
```

---

## Shop Prices

```json
{
    "PRICES": {
        "titles": {"champion": 500, "legendary": 1000, "epic": 300},
        "clan": {"create": 5000, "upgrade_slots": 2000, "banner": 1000},
        "keys": {"chestkeydraw": 200, "chestkeymoney": 500},
        "pets": {"rename": 100, "premium_food": 50}
    }
}
```

---

## Chests

```json
{
    "CHEST_CONFIG": {
        "draw": {
            "rewards": {
                "common": {"weight": 60, "items": ["credits_50", "credits_100"]},
                "uncommon": {"weight": 30, "items": ["credits_250"]},
                "rare": {"weight": 8, "items": ["credits_500", "chestkeydraw_1"]},
                "epic": {"weight": 2, "items": ["credits_1000", "title_epic"]}
            }
        },
        "money": {
            "rewards": {
                "common": {"weight": 40, "items": ["credits_500"]},
                "uncommon": {"weight": 35, "items": ["credits_1000"]},
                "rare": {"weight": 20, "items": ["credits_2500"]},
                "epic": {"weight": 5, "items": ["credits_5000"]}
            }
        }
    }
}
```

---

## Events (`event.json`)

```json
[
    {
        "id": "event_001",
        "name": "Double ELO Weekend",
        "start": "2024-03-01T00:00:00Z",
        "end": "2024-03-03T23:59:59Z",
        "class_allowed": null,
        "elo_multiplier": 2.0,
        "xp_multiplier": 1.0,
        "credits_multiplier": 1.5
    }
]
```

---

## Titles (`titles.json`)

```json
{
    "champion": {
        "name": "Champion",
        "description": "Reach 2000 ELO",
        "rarity": "rare",
        "condition": {"type": "elo", "value": 2000}
    }
}
```

---

## Emoji

```json
{
    "EMOJI": {
        "rank": {
            "bronze": "🥉", "silver": "🥈", "gold": "🥇",
            "platinum": "💎", "diamond": "💠",
            "champion": "🏆", "grandmaster": "👑"
        },
        "format": {"1": "1️⃣", "2": "2️⃣", "3": "3️⃣", "4": "4️⃣", "5": "5️⃣", "6": "6️⃣"},
        "currency": "🪙",
        "chest": "📦",
        "key": "🗝️",
        "pet": "🐾"
    }
}
```

---

## Translation (`translation/`)

Files: `french.json`, `english.json`, `spanish.json`, `portuguese.json`

```json
{
    "commands": {
        "profile": {
            "title": "Player Profile",
            "elo": "ELO",
            "wins": "Wins"
        }
    },
    "queue": {
        "join": "Join Queue",
        "cancel": "Cancel"
    },
    "errors": {
        "not_registered": "You must register first!",
        "not_enough_credits": "Not enough credits!"
    }
}
```

| Code | Language |
|------|----------|
| 0 | French |
| 1 | English |
| 2 | Spanish |
| 3 | Portuguese |

---

## Accessing Configuration

```python
from config import Config
Config.load("alpha")

token = Config.get("DISCORD_TOKEN")
season = Config.get("SEASON", {}).get("number")
```
