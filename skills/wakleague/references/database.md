# WakLeague - Database Models

## DBHandling (`dbhand.py`)

Thread-safe database wrapper around APSW.

```python
class DBHandling:
    def execute(self, query, params=())      # Write
    def fetchone(self, query, params=())     # Single row
    def fetchall(self, query, params=())     # All rows
```

---

## WakMMUser (`user.py`)

Player data model with per-format, per-server ELO tracking.

### Properties

| Field | Type | Description |
|-------|------|-------------|
| `discordid` | TEXT PK | Discord user ID |
| `pseudo` | TEXT | Wakfu character name |
| `elo{1-6}{1-4}` | INTEGER | ELO per format/server |
| `winstreak{1-6}{1-4}` | INTEGER | Win streaks |
| `nbrmatch{1-6}{1-4}` | INTEGER | Match count |
| `nbvict{1-6}{1-4}` | INTEGER | Win count |
| `elo_max` | INTEGER | Historical max ELO |
| `credits` | INTEGER | WakLingots currency |
| `chestkeydraw` | INTEGER | Draw chest keys |
| `chestkeymoney` | INTEGER | Money chest keys |
| `titles` | TEXT | JSON array of titles |
| `titleused` | TEXT | Current title |
| `clan` | INTEGER | Clan ID |
| `language` | INTEGER | 0=FR, 1=EN, 2=ES, 3=PT |
| `level` | INTEGER | Wakfu level |
| `classe` | TEXT | Character class |
| `register` | INTEGER | Registration status |

### Key Methods

```python
def getELO(format, server) -> int
def updateELO(format, server, delta)
def getWinStreak(format, server) -> int
def getWinRate(format, server) -> float
def addToQueue(format, server, mode, lvl_range, mates=[])
def removeFromQueue()
def isInQueue() -> bool
def getQueueData() -> dict
def addCredits(amount)
def removeCredits(amount)
def addTitle(title_id)
def setTitle(title_id)
def setLanguage(lang)
```

---

## WakMMMatch (`match.py`)

Match data model for tracking match lifecycle.

### Properties

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER PK | Match ID |
| `ispending` | INTEGER | 1=pending, 0=finished |
| `queue` | INTEGER | Server (1-4) |
| `format` | INTEGER | Format (1-6) |
| `mode` | INTEGER | 0=Ranked, 1=Chill |
| `usr1id` | TEXT | Team 1 leader |
| `usr2id` | TEXT | Team 2 leader |
| `usr1m1id...usr1m5id` | TEXT | Team 1 members |
| `usr2m1id...usr2m5id` | TEXT | Team 2 members |
| `usr1classe...usr2m5classe` | TEXT | Character classes |
| `usr1pseudo...usr2m5pseudo` | TEXT | Character pseudos |
| `discordchannel` | TEXT | Match channel ID |
| `result` | INTEGER | 1/2=winner, -1=cancelled |
| `deltaelo1/deltaelo2` | INTEGER | ELO changes |

### Team Player Mapping

Format N → Teams of N players:
- Team 1: `usr1id`, `usr1m1id` ... `usr1m{N-1}id`
- Team 2: `usr2id`, `usr2m1id` ... `usr2m{N-1}id`

### Key Methods

```python
def createNewMatch(players, format, server, mode) -> WakMMMatch
def getTeam1() -> list[WakMMUser]
def getTeam2() -> list[WakMMUser]
def getAllPlayers() -> list[WakMMUser]
def setPlayerClass(player_id, classe)
def setPlayerPseudo(player_id, pseudo)
def playerConfirmed(player_id)
def allPlayersConfirmed() -> bool
def addResult(result)
def concludeMatch()
def cancelMatch()
def isExpired() -> bool
```

---

## WakMMClan (`clan.py`)

Clan data model.

### Properties

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER PK | Clan ID |
| `owner` | TEXT | Owner Discord ID |
| `name` | TEXT | Clan name |
| `credits` | INTEGER | Shared currency |
| `score` | INTEGER | Computed score |
| `hasemoji` | INTEGER | Custom emoji flag |

### Key Methods

```python
def createClan(owner_id, name) -> WakMMClan
def addMember(user_id)
def removeMember(user_id)
def getMembers() -> list[WakMMUser]
def getScore() -> int
def updateScore()
def addCredits(amount)
def removeCredits(amount)
def invitePlayer(user_id)
def acceptInvite(user_id)
def declineInvite(user_id)
def transferOwnership(new_owner_id)
def deleteClan()
```

---

## Queue Table (Temporary)

```sql
CREATE TABLE queue (
    id TEXT PRIMARY KEY,
    lvl TEXT,           -- JSON [min, max]
    queue INTEGER,      -- Server 1-4
    format INTEGER,     -- Format 1-6
    mode INTEGER,       -- 0=Ranked, 1=Chill
    mate1 TEXT,         -- Teammate 1
    mate2 TEXT,
    mate3 TEXT,
    mate4 TEXT,
    mate5 TEXT,
    time INTEGER        -- Timestamp
);
```

---

## Draft (`draft.py`)

Pick-ban system for 6v6 tournaments.

### Properties

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER PK | Draft ID |
| `channel_id` | TEXT | Discord channel |
| `captain1` | TEXT | Team 1 captain |
| `captain2` | TEXT | Team 2 captain |
| `players1/players2` | TEXT | JSON arrays |
| `banned/picked` | TEXT | JSON arrays |
| `current_pick` | INTEGER | Whose turn |
| `phase` | INTEGER | 0=ban, 1=pick |
| `status` | INTEGER | 0=waiting, 1=active, 2=done |

### Key Methods

```python
def create(captain1, captain2, players1, players2) -> Draft
def addBan(player_id, character)
def addPick(player_id, character)
def nextTurn()
def getCurrentCaptain() -> str
def isComplete() -> bool
def cancel()
```

---

## Group (`group.py`)

Team queue management.

```python
class Group:
    def create(leader_id, member_ids, format, server, mode, lvl_range)
    def addMember(user_id)
    def removeMember(user_id)
    def isFull() -> bool
    def getMembers() -> list[str]
    def delete()
```

---

## Query Patterns

```python
# Get user
db.fetchone("SELECT * FROM users WHERE discordid = ?", (discord_id,))

# Get queued users
db.fetchall("SELECT * FROM queue")

# Get pending matches
db.fetchall("SELECT * FROM matches WHERE ispending = 1")

# Get clan members
db.fetchall("SELECT * FROM users WHERE clan = ?", (clan_id,))

# Update ELO
db.execute("UPDATE users SET elo1_1 = ? WHERE discordid = ?", (new_elo, discord_id))
```
