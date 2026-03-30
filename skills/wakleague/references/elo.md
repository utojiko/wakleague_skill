# WakLeague - ELO System

## Core Parameters (`elo_utils.py`)

```python
ELO_BASE = 125                    # Base K-factor
MAX_K_FACTOR = 100                # Maximum K-factor
MIN_K_FACTOR = 40                 # Minimum K-factor
ELO_STAGE_K_FACTOR = 2000         # ELO threshold for reduced K

WINSTREAK_STAGE1 = 3              # 1.3x bonus at 3 wins
WINSTREAK_STAGE2 = 7              # 1.6x bonus at 7 wins
BONUS_FIRST_WIN = 2               # First win of day multiplier

DECAY_THRESHOLD_DAYS = 30         # Days before decay
DECAY_AMOUNT = 10                 # ELO lost per decay
```

---

## K-Factor Calculation

K-factor determines ELO change magnitude. Scales with player ELO and win streaks.

```python
def getKFactor(elo: int, win_streak: int) -> float:
    k = ELO_BASE  # 125
    
    # Reduce for high ELO players
    if elo >= ELO_STAGE_K_FACTOR:  # 2000
        reduction = min(elo - ELO_STAGE_K_FACTOR, 3000) / 50
        k = max(MAX_K_FACTOR - reduction, MIN_K_FACTOR)  # 100 to 40
    
    # Win streak bonus
    if win_streak >= WINSTREAK_STAGE2:  # 7+
        k *= WINSTREAK_MULTIPLIER2  # 1.6x
    elif win_streak >= WINSTREAK_STAGE1:  # 3+
        k *= WINSTREAK_MULTIPLIER1  # 1.3x
    
    return k
```

### K-Factor Examples

| ELO | Streak | K-Factor |
|-----|--------|----------|
| 1000 | 0 | 125 |
| 1500 | 0 | 125 |
| 2000 | 0 | 100 |
| 2200 | 0 | 60 |
| 2400 | 0 | 40 |
| 1500 | 3 | 162.5 |
| 1500 | 7 | 200 |

---

## Expected Score

Probability of winning based on ELO difference.

```python
def expectedScore(elo_a: int, elo_b: int) -> float:
    return 1 / (1 + 10 ** ((elo_b - elo_a) / 400))
```

### Expected Score Table

| ELO Diff | Lower ELO | Higher ELO |
|----------|-----------|------------|
| 0 | 50% | 50% |
| 100 | 46% | 54% |
| 200 | 42% | 58% |
| 400 | 36% | 64% |
| 800 | 24% | 76% |

---

## ELO Change Calculation

```python
def calculateEloChange(winner_elo, loser_elo, winner_streak, loser_streak, is_first_win=False):
    expected = expectedScore(winner_elo, loser_elo)
    delta = getKFactor(winner_elo, winner_streak) * (1 - expected)
    
    if is_first_win:
        delta *= BONUS_FIRST_WIN  # 2x
    
    return round(delta), round(-delta)
```

### Example

```
Winner: 1500 ELO, 0 streak
Loser: 1500 ELO, 0 streak
Expected: 0.5
K: 125
Delta: 125 × (1 - 0.5) = 62.5 → 63

Result: Winner +63, Loser -63
```

---

## Win Streak System

```python
def incrementStreak(user_id, format, server):
    streak = user.getWinStreak(format, server) + 1
    user.setWinStreak(format, server, streak)
    
    if streak == 3:  unlockAchievement(user_id, "streak_3")
    if streak == 7:  unlockAchievement(user_id, "streak_7")
    if streak == 15: unlockAchievement(user_id, "streak_15")

def resetStreak(user_id, format, server):
    user.setWinStreak(format, server, 0)
```

### Streak Bonuses

| Streak | Multiplier | K-Factor (at 1500) |
|--------|------------|---------------------|
| 0-2 | 1.0x | 125 |
| 3-6 | 1.3x | 162.5 |
| 7+ | 1.6x | 200 |

---

## First Win of Day

```python
def checkFirstWin(user_id) -> bool:
    user = WakMMUser(user_id)
    if user.getLastFirstWinDate() != getServerDate():
        user.setLastFirstWinDate(getServerDate())
        return True
    return False
```

**Effect:** 2x ELO gain for first win of the day.

---

## ELO Decay

Inactive players lose ELO over time.

```python
def applyDecay(user_id):
    user = WakMMUser(user_id)
    days_inactive = user.getDaysSinceLastMatch()
    
    if days_inactive < DECAY_THRESHOLD_DAYS:  # 30 days
        return
    
    for format in range(1, 7):
        for server in range(1, 5):
            current_elo = user.getELO(format, server)
            if current_elo > DECAY_MIN_ELO:  # 1000
                user.updateELO(format, server, -DECAY_AMOUNT)  # -10
```

---

## Rank Tiers

| Rank | ELO Range | Emoji |
|------|-----------|-------|
| Bronze | 0-1399 | 🥉 |
| Silver | 1400-1899 | 🥈 |
| Gold | 1900-2399 | 🥇 |
| Platinum | 2400-2899 | 💎 |
| Diamond | 2900-3399 | 💠 |
| Champion | 3400-3499 | 🏆 |
| Grandmaster | 3500+ | 👑 |

### Rank Function

```python
def getRank(elo: int) -> tuple[str, str]:
    ranks = [
        ("Bronze", 0, 1399),
        ("Silver", 1400, 1899),
        ("Gold", 1900, 2399),
        ("Platinum", 2400, 2899),
        ("Diamond", 2900, 3399),
        ("Champion", 3400, 3499),
        ("Grandmaster", 3500, float('inf')),
    ]
    for name, min_elo, max_elo in ranks:
        if min_elo <= elo <= max_elo:
            return name, getRankEmoji(name)
    return "Unranked", "⬜"
```

---

## Per-Format Storage

ELO stored separately per format and server.

```python
# Columns: elo1_1, elo1_2, elo1_3, elo1_4 (1v1 per server)
#          elo2_1, elo2_2, ... (2v2 per server)
#          ...

user.getELO(format=3, server=2)  # Returns elo3_2
```

---

## Match ELO Application

```python
def applyMatchResult(match: WakMMMatch, result: int):
    team1 = match.getTeam1()
    team2 = match.getTeam2()
    
    # Average ELO for teams
    avg_elo_1 = sum(u.getELO(match.format, match.queue) for u in team1) / len(team1)
    avg_elo_2 = sum(u.getELO(match.format, match.queue) for u in team2) / len(team2)
    
    # Calculate delta
    delta = calculateEloChange(
        winner_elo=avg_elo_1 if result == 1 else avg_elo_2,
        loser_elo=avg_elo_2 if result == 1 else avg_elo_1,
        winner_streak=team1[0].getWinStreak() if result == 1 else team2[0].getWinStreak(),
        loser_streak=team2[0].getWinStreak() if result == 1 else team1[0].getWinStreak(),
        is_first_win=checkFirstWin(winner.id)
    )
    
    # Apply
    winners = team1 if result == 1 else team2
    losers = team2 if result == 1 else team1
    
    for player in winners:
        player.updateELO(match.format, match.queue, delta[0])
        player.incrementWinStreak(match.format, match.queue)
    
    for player in losers:
        player.updateELO(match.format, match.queue, delta[1])
        player.resetWinStreak(match.format, match.queue)
```

---

## Display Formatting

```python
def formatElo(elo: int) -> str:
    return f"{elo:,}"

def formatEloChange(delta: int) -> str:
    if delta > 0: return f"+{delta} ▲"
    if delta < 0: return f"{delta} ▼"
    return "0"
```

Example: `ELO: 2,347 +25 ▲`
