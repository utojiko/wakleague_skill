# WakLeague - Views (Discord UI)

## Base Pattern

```python
import discord
from discord import ui

class MyView(ui.View):
    def __init__(self, ctx_data: dict = None):
        super().__init__(timeout=300)
        self.ctx_data = ctx_data or {}
    
    @ui.button(label="Action", style=discord.ButtonStyle.green)
    async def action_button(self, interaction, button):
        await interaction.response.edit_message(content="Updated!")
    
    @ui.select(cls=ui.Select, placeholder="Choose...",
                options=[discord.SelectOption(label="Opt 1", value="1")])
    async def select_callback(self, interaction, select):
        await interaction.response.edit_message(content=f"Selected: {select.values[0]}")
    
    async def on_timeout(self):
        for child in self.children:
            child.disabled = True
        await self.message.edit(view=self)
```

---

## Queue Views (`views/queue.py`)

### QueueView

```python
class QueueView(ui.View):
    def __init__(self):
        super().__init__(timeout=None)
    
    @ui.button(label="Join Queue", style=discord.ButtonStyle.green, custom_id="join_queue")
    async def join_queue(self, interaction, button):
        await interaction.response.send_modal(QueueOptionsModal())
    
    @ui.button(label="Rematch", style=discord.ButtonStyle.blurple, custom_id="rematch")
    async def rematch(self, interaction, button):
        user = WakMMUser(interaction.user.id)
        if user.isInQueue():
            await interaction.response.send_message("Already in queue!", ephemeral=True)
            return
        # Re-queue with previous settings
    
    @ui.button(label="Cancel", style=discord.ButtonStyle.red, custom_id="cancel_queue")
    async def cancel(self, interaction, button):
        user = WakMMUser(interaction.user.id)
        user.removeFromQueue()
```

### QueueOptionsModal

```python
class QueueOptionsModal(ui.Modal, title="Queue Options"):
    format_select = ui.TextInput(label="Format (1-6)", placeholder="1")
    server_select = ui.TextInput(label="Server (1-4)", placeholder="1")
    mode_select = ui.TextInput(label="Mode (0=Ranked, 1=Chill)", placeholder="0")
    lvl_min = ui.TextInput(label="Min Level", placeholder="200")
    lvl_max = ui.TextInput(label="Max Level", placeholder="250")
    
    async def callback(self, interaction):
        user = WakMMUser(interaction.user.id)
        user.addToQueue(
            format=int(self.format_select.value),
            server=int(self.server_select.value),
            mode=int(self.mode_select.value),
            lvl_range=[int(self.lvl_min.value), int(self.lvl_max.value)]
        )
```

### QueueGroupSelectView

```python
class QueueGroupSelectView(ui.View):
    @ui.select(cls=ui.UserSelect, min_values=1, max_values=5, placeholder="Select teammates")
    async def select_teammates(self, interaction, select):
        self.selected_members = [u.id for u in select.values]
```

---

## Registration Views (`views/register.py`)

### LanguageView

```python
class LanguageView(ui.View):
    @ui.button(label="🇫🇷 Français", style=discord.ButtonStyle.secondary, custom_id="lang_fr")
    async def french(self, interaction, button):
        await self.start_registration(interaction, lang=0)
    
    @ui.button(label="🇬🇧 English", style=discord.ButtonStyle.secondary, custom_id="lang_en")
    async def english(self, interaction, button):
        await self.start_registration(interaction, lang=1)
```

### RegisterView + RegistrationModal

```python
class RegistrationModal(ui.Modal, title="Character Registration"):
    pseudo = ui.TextInput(label="Character Name", placeholder="Your Wakfu name")
    level = ui.TextInput(label="Character Level", placeholder="245")
    character_class = ui.TextInput(label="Main Class", placeholder="Eliotrope")
    
    async def callback(self, interaction):
        user = WakMMUser(interaction.user.id)
        user.pseudo = self.pseudo.value
        user.level = int(self.level.value)
        user.classe = self.character_class.value
        user.register = 1
```

---

## Match Views (`views/match_found.py`, `match_result.py`)

### ClassAndPseudoView

```python
class ClassAndPseudoView(ui.View):
    def __init__(self, match_id: int):
        self.match_id = match_id
    
    @ui.select(cls=ui.Select, placeholder="Select your class")
    async def select_class(self, interaction, select):
        self.selected_class = select.values[0]
    
    @ui.button(label="Confirm", style=discord.ButtonStyle.green)
    async def confirm(self, interaction, button):
        if not self.selected_class:
            await interaction.response.send_message("Select a class first!", ephemeral=True)
            return
        
        match = WakMMMatch(self.match_id)
        match.setPlayerClass(interaction.user.id, self.selected_class)
        
        if match.allPlayersConfirmed():
            await match.start()
```

### MatchResultView

```python
class MatchResultView(ui.View):
    def __init__(self, match_id: int):
        self.match_id = match_id
    
    @ui.button(label="Victory", style=discord.ButtonStyle.green)
    async def victory(self, interaction, button):
        match = WakMMMatch(self.match_id)
        if interaction.user.id in match.getTeam1():
            match.addResult(1)
        else:
            match.addResult(2)
    
    @ui.button(label="Defeat", style=discord.ButtonStyle.red)
    async def defeat(self, interaction, button):
        match = WakMMMatch(self.match_id)
        if interaction.user.id in match.getTeam1():
            match.addResult(2)
        else:
            match.addResult(1)
    
    @ui.button(label="Dispute", style=discord.ButtonStyle.gray)
    async def dispute(self, interaction, button):
        match = WakMMMatch(self.match_id)
        match.flagForReview()
        await notify_referees(match)
```

---

## Profile Views (`views/profile.py`)

### ProfileView

```python
class ProfileView(ui.View):
    def __init__(self, user_id: int, page: int = 0):
        self.user_id = user_id
        self.page = page
    
    @ui.button(label="◀", style=discord.ButtonStyle.secondary)
    async def prev_page(self, interaction, button):
        self.page -= 1
        await self.refresh(interaction)
    
    @ui.button(label="▶", style=discord.ButtonStyle.secondary)
    async def next_page(self, interaction, button):
        self.page += 1
        await self.refresh(interaction)
```

### TitlesView

```python
class TitlesView(ui.View):
    @ui.select(cls=ui.Select, placeholder="Select a title")
    async def select_title(self, interaction, select):
        self.selected_title = select.values[0]
    
    @ui.button(label="Equip", style=discord.ButtonStyle.green)
    async def equip(self, interaction, button):
        user = WakMMUser(self.user_id)
        user.setTitle(self.selected_title)
```

---

## Clan Views (`views/clan_*.py`)

### ClanView

```python
class ClanView(ui.View):
    @ui.button(label="Create Clan", style=discord.ButtonStyle.green, custom_id="clan_create")
    async def create(self, interaction, button):
        await interaction.response.send_modal(CreateClanModal())
    
    @ui.button(label="Join Clan", style=discord.ButtonStyle.blurple, custom_id="clan_join")
    async def join(self, interaction, button):
        await interaction.response.send_modal(JoinClanModal())
    
    @ui.button(label="Leave Clan", style=discord.ButtonStyle.red, custom_id="clan_leave")
    async def leave(self, interaction, button):
        clan = WakMMClan(user.clan)
        clan.removeMember(interaction.user.id)
```

### ClanOwnerView

```python
class ClanOwnerView(ui.View):
    @ui.button(label="Invite", style=discord.ButtonStyle.green)
    async def invite(self, interaction, button):
        await interaction.response.send_modal(InviteModal())
    
    @ui.button(label="Kick", style=discord.ButtonStyle.red)
    async def kick(self, interaction, button):
        await interaction.response.send_modal(KickModal())
```

---

## Chest Views (`views/chest.py`)

```python
class ChestView(ui.View):
    def __init__(self, chest_type: str):
        self.chest_type = chest_type
    
    @ui.button(label="Open Draw Chest", style=discord.ButtonStyle.green)
    async def open_draw(self, interaction, button):
        user = WakMMUser(interaction.user.id)
        if user.chestkeydraw < 1:
            await interaction.response.send_message("No keys!", ephemeral=True)
            return
        
        user.chestkeydraw -= 1
        reward = rollChestReward(self.chest_type)
        user.giveReward(reward)
```

---

## Pet Views (`views/pet.py`)

```python
class PetView(ui.View):
    @ui.button(label="Feed", style=discord.ButtonStyle.green)
    async def feed(self, interaction, button):
        user = WakMMUser(self.user_id)
        pet = user.getPet()
        
        if user.credits < 100:
            await interaction.response.send_message("Need 100 credits!", ephemeral=True)
            return
        
        user.credits -= 100
        pet.happiness = min(100, pet.happiness + 10)
```

---

## Button Styles

```python
discord.ButtonStyle.primary    # Blurple
discord.ButtonStyle.secondary # Gray
discord.ButtonStyle.success   # Green
discord.ButtonStyle.danger    # Red
```

---

## Response Types

```python
# Edit original
await interaction.response.edit_message(embed=embed, view=view)

# Send new
await interaction.response.send_message("text")

# Ephemeral (private)
await interaction.response.send_message("private", ephemeral=True)

# Modal
await interaction.response.send_modal(MyModal())

# Defer (for long ops)
await interaction.response.defer()
await interaction.edit_original_response(content="Done")
```
