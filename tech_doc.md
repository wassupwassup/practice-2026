# Техническое руководство по созданию 2D RPG на Python и Pygame

## Оглавление
1. [Техническое руководство (туториал) для начинающих]
   - 1.1. [Создание игрового окна и цикла]
   - 1.2. [Система частиц]
   - 1.3. [Загрузка спрайтов]
   - 1.4. [Класс игрока (Player)]
   - 1.5. [Класс врага (Enemy)]
   - 1.6. [Класс NPC]
   - 1.7. [Класс предмета (Item)]
   - 1.8. [Диалоговая система (DialogueBox)]
   - 1.9. [Боевая система (CombatSystem)]
   - 1.10. [Экраны победы и поражения]
   - 1.11. [Главный класс Game]
2. [Заключение]

---

# 1. Техническое руководство (туториал) для начинающих

Это руководство проведёт вас через создание 2D RPG на Python с использованием Pygame. В качестве основы используется игра **"Изгнание нежити"** — проект, в котором реализованы: движение по карте, боевая система, NPC, предметы, система уровней и маны.

## 1.1. Создание игрового окна и цикла

```python
import pygame
import sys

pygame.init()
SCREEN_WIDTH = 1024
SCREEN_HEIGHT = 768
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Изгнание нежити")
clock = pygame.time.Clock()
FPS = 60

running = True
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill((30, 30, 50))
    pygame.display.flip()
    clock.tick(FPS)

pygame.quit()
sys.exit()
```
## 1.2 Система частиц
```python
class ParticleSystem:
    def __init__(self):
        self.particles = []

    def emit(self, x, y, color, count=10):
        for _ in range(count):
            self.particles.append({
                'x': x, 'y': y,
                'vx': random.uniform(-100, 100),
                'vy': random.uniform(-100, 100),
                'life': 1.0,
                'color': color
            })

    def update(self, dt):
        for p in self.particles[:]:
            p['x'] += p['vx'] * dt
            p['y'] += p['vy'] * dt
            p['life'] -= dt * 2
            if p['life'] <= 0:
                self.particles.remove(p)

    def draw(self, screen):
        for p in self.particles:
            size = int(6 * p['life'])
            pygame.draw.circle(screen, p['color'][:3], (int(p['x']), int(p['y'])), max(1, size))
```
## 1.3 Загрузка спрайтов 

```python
def load_sprite(path, size):
    if path and os.path.exists(path):
        try:
            sprite = pygame.image.load(path).convert_alpha()
            return pygame.transform.scale(sprite, size)
        except Exception as e:
            print(f" Не удалось загрузить {path}: {e}")
    raise FileNotFoundError(f" нет спрайта: {path}")

# Размеры спрайтов
SPRITE_SIZE_MAP = 96      # Размер спрайтов на карте
SPRITE_SIZE_COMBAT = 192  # Размер спрайтов в бою
ITEM_SIZE = 48            # Размер предметов
```

## 1.4 Класс игрока

```python
class Player:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.name = "Герой"
        self.level = 1
        self.max_hp = 120
        self.hp = 120
        self.max_mp = 80
        self.mp = 80
        self.strength = 15
        self.defense = 10
        self.exp = 0
        self.exp_to_next = 100
        self.gold = 500
        self.potions = 5
        self.equipment = {'weapon': None, 'armor': None}
        self.rect = pygame.Rect(x, y, SPRITE_SIZE_MAP, SPRITE_SIZE_MAP)

        self.sprite = load_sprite(PLAYER_SPRITE_PATH, (SPRITE_SIZE_MAP, SPRITE_SIZE_MAP))
        self.combat_sprite = load_sprite(PLAYER_SPRITE_PATH, (SPRITE_SIZE_COMBAT, SPRITE_SIZE_COMBAT))
        self.move_speed = 300

    def get_damage(self):
        dmg = self.strength
        if self.equipment['weapon']:
            dmg += self.equipment['weapon'].get('damage', 0)
        return dmg

    def take_damage(self, dmg):
        actual = max(1, dmg - self.defense)
        self.hp -= actual
        if self.hp < 0:
            self.hp = 0
        return actual

    def heal(self, amount):
        self.hp = min(self.max_hp, self.hp + amount)

    def is_alive(self):
        return self.hp > 0

    def gain_exp(self, amount):
        self.exp += amount
        if self.exp >= self.exp_to_next:
            self.level_up()

    def level_up(self):
        self.level += 1
        self.exp -= self.exp_to_next
        self.exp_to_next = int(self.exp_to_next * 1.2)
        self.max_hp += 25
        self.hp = self.max_hp
        self.max_mp += 15
        self.mp = self.max_mp
        self.strength += 4
        self.defense += 2
```
## 1.5 Класс Enemy

```python
class Enemy:
    def __init__(self, x, y, enemy_type, level=1):
        self.x = x
        self.y = y
        self.type = enemy_type
        self.level = level

        types = {
            'skeleton': {'name': 'Скелет', 'hp': 65, 'damage': 18, 'exp': 55, 'gold': 35},
            'ghoul': {'name': 'Мертвец', 'hp': 333, 'damage': 15, 'exp': 75, 'gold': 10},
            'lich': {'name': 'Лич', 'hp': 666, 'damage': 33, 'exp': 777, 'gold': 1000},
        }

        info = types.get(enemy_type, types['skeleton'])
        self.name = info['name']
        self.max_hp = info['hp'] + (level - 1) * 15
        self.hp = self.max_hp
        self.damage = info['damage'] + (level - 1) * 3
        self.exp_reward = info['exp'] + (level - 1) * 10
        self.gold_reward = info['gold'] + (level - 1) * 5
        self.rect = pygame.Rect(x, y, SPRITE_SIZE_MAP, SPRITE_SIZE_MAP)

        self.sprite = load_sprite(ENEMY_SPRITES[enemy_type], (SPRITE_SIZE_MAP, SPRITE_SIZE_MAP))
        self.combat_sprite = load_sprite(ENEMY_SPRITES[enemy_type], (SPRITE_SIZE_COMBAT, SPRITE_SIZE_COMBAT))

    def attack(self):
        return random.randint(self.damage - 5, self.damage + 5)

    def take_damage(self, dmg):
        self.hp -= dmg
        if self.hp < 0:
            self.hp = 0
        return dmg

    def is_alive(self):
        return self.hp > 0
```

## 1.6 Класс NPC

```python
class NPC:
    def __init__(self, x, y, npc_id):
        self.x = x
        self.y = y
        self.id = npc_id
        self.rect = pygame.Rect(x, y, SPRITE_SIZE_MAP, SPRITE_SIZE_MAP)
        self.dialogues = self.get_dialogues()
        self.current_dialogue = 0
        self.sprite = load_sprite(NPC_SPRITES[npc_id], (SPRITE_SIZE_MAP, SPRITE_SIZE_MAP))

    def get_dialogues(self):
        dialogues = {
            'elder': ["Приветствую, славный воин!", "Нам нужна твоя помощь!", 
                      "Вокруг много опасных монстров.", "Нужно очистить лес от этой нежити!"],
            'merchant': ["Добро пожаловать!", "Могу продать зелья за 30 золота.", "Нажми E, чтобы купить."],
            'healer': ["Я могу исцелить тебя.", "Лечение стоит 20 золота.", "Хочешь восстановить здоровье?"]
        }
        return dialogues.get(self.id, ["Привет!", "Чем могу помочь?"])

    def get_next_dialogue(self):
        text = self.dialogues[self.current_dialogue]
        self.current_dialogue = (self.current_dialogue + 1) % len(self.dialogues)
        return text
```

## 1.7 Класс Предмета (Item)

```python
class Item:
    def __init__(self, x, y, item_type):
        self.x = x
        self.y = y
        self.type = item_type
        self.rect = pygame.Rect(x, y, ITEM_SIZE, ITEM_SIZE)
        self.sprite = self.create_sprite()

        if item_type == 'potion':
            self.name = "Зелье здоровья"
            self.value = 50
        elif item_type == 'gold':
            self.name = "Золото"
            self.value = random.randint(10, 50)
        elif item_type == 'sword':
            self.name = "Стальной меч"
            self.value = {"damage": 12}

    def create_sprite(self):
        sprite = pygame.Surface((ITEM_SIZE, ITEM_SIZE), pygame.SRCALPHA)
        s = ITEM_SIZE

        if self.type == 'potion':
            pygame.draw.rect(sprite, (255, 50, 50), (s//4, s//4, s//2, s//1.6))
            pygame.draw.circle(sprite, (255, 50, 50), (s//2, s//5), s//8)
            pygame.draw.rect(sprite, (200, 200, 200), (s//2.3, s//4, s//8, s//1.6))
        elif self.type == 'gold':
            pygame.draw.ellipse(sprite, (255, 215, 0), (s//4, s//3, s//2, s//4))
            pygame.draw.circle(sprite, (255, 215, 0), (s//2, s//2), s//5)
        elif self.type == 'sword':
            pygame.draw.line(sprite, (200, 200, 250), (s//1.3, s//4), (s//4, s//1.3), 5)
            pygame.draw.rect(sprite, (150, 100, 50), (s//2.7, s//1.7, s//4, s//4))
        return sprite
```

## 1.8 Диалоговая система

```python
class DialogueBox:
    def __init__(self, screen):
        self.screen = screen
        self.active = False
        self.lines = []
        self.current_line = 0
        self.display_text = ""
        self.char_index = 0
        self.char_timer = 0
        self.font = pygame.font.Font(None, 28)

    def start(self, dialogues):
        self.active = True
        self.lines = dialogues
        self.current_line = 0
        self.display_text = ""
        self.char_index = 0
        self.next_line()

    def next_line(self):
        if self.current_line < len(self.lines):
            self.display_text = ""
            self.char_index = 0
            self.target_text = self.lines[self.current_line]
            self.current_line += 1
        else:
            self.active = False

    def update(self, dt):
        if not self.active:
            return
        self.char_timer += dt
        if self.char_timer > 0.03 and self.char_index < len(self.target_text):
            self.char_timer = 0
            self.char_index += 1
            self.display_text = self.target_text[:self.char_index]

    def handle_event(self, event):
        if not self.active:
            return False
        if event.type == KEYDOWN and event.key == K_SPACE:
            if self.char_index < len(self.target_text):
                self.char_index = len(self.target_text)
                self.display_text = self.target_text
            else:
                self.next_line()
            return True
        return False

    def draw(self):
        if not self.active:
            return
        dialog_rect = pygame.Rect(50, SCREEN_HEIGHT - 180, SCREEN_WIDTH - 100, 140)
        pygame.draw.rect(self.screen, COLORS['UI'], dialog_rect)
        pygame.draw.rect(self.screen, COLORS['TEXT_DARK'], dialog_rect, 3)
        # ... отрисовка текста с переносом строк
```
## 1.9 Боевая система

```python
class CombatSystem:
    def __init__(self, screen, player, enemy, particles):
        self.screen = screen
        self.player = player
        self.enemy = enemy
        self.particles = particles
        self.player_turn = True
        self.message = ""
        self.message_timer = 0
        self.selected_action = 0
        self.combat_active = True
        self.font = pygame.font.Font(None, 36)
        self.small_font = pygame.font.Font(None, 24)
        self.combat_bg = self.load_combat_background()

    def handle_event(self, event):
        if not self.combat_active:
            return
        if event.type == KEYDOWN and self.player_turn:
            if event.key == K_UP:
                self.selected_action = (self.selected_action - 1) % 3
            elif event.key == K_DOWN:
                self.selected_action = (self.selected_action + 1) % 3
            elif event.key == K_RETURN or event.key == K_SPACE:
                if self.selected_action == 0:
                    self.player_attack()
                elif self.selected_action == 1:
                    self.player_skill()
                elif self.selected_action == 2:
                    self.player_heal()

    def player_attack(self):
        damage = self.player.get_damage()
        actual = self.enemy.take_damage(damage)
        self.message = f" Вы нанесли {actual} урона!"
        self.message_timer = 1.5
        self.particles.emit(self.enemy.x + 48, self.enemy.y + 48, COLORS['HEALTH'], 15)
        self.player_turn = False

    def player_skill(self):
        if self.player.mp >= 15:
            damage = self.player.get_damage() * 2
            actual = self.enemy.take_damage(damage)
            self.player.mp -= 15
            self.message = f" Мощный удар! {actual} урона!"
            self.particles.emit(self.enemy.x + 48, self.enemy.y + 48, (255,100,100), 25)
        else:
            self.message = " Недостаточно маны!"
            self.message_timer = 1.0
            return
        self.player_turn = False

    def player_heal(self):
        if self.player.potions > 0:
            heal_amount = 50
            self.player.heal(heal_amount)
            self.player.potions -= 1
            self.message = f" Вы восстановили {heal_amount} HP!"
            self.particles.emit(self.player.x + 48, self.player.y + 48, COLORS['HEALTH'], 20)
        else:
            self.message = "Нет зелий!"
            self.message_timer = 1.0
            return
        self.player_turn = False

    def enemy_attack(self):
        damage = self.enemy.attack()
        actual = self.player.take_damage(damage)
        self.message = f" {self.enemy.name} нанес {actual} урона!"
        self.message_timer = 1.5
        self.particles.emit(self.player.x + 48, self.player.y + 48, COLORS['HEALTH'], 15)
        self.player_turn = True

    def update(self, dt):
        if self.message_timer > 0:
            self.message_timer -= dt
        if not self.player_turn and self.enemy.is_alive() and self.player.is_alive():
            self.enemy_attack()
        if not self.enemy.is_alive() or not self.player.is_alive():
            self.combat_active = False
            return True
        return False
```
## 1.10 Экран победы и поражения

```python
class VictoryScreen:
    def __init__(self, screen):
        self.screen = screen
        self.active = False
        self.timer = 0
        self.font_big = pygame.font.Font(None, 72)
        self.font_small = pygame.font.Font(None, 36)
        self.particles = ParticleSystem()

    def start(self):
        self.active = True
        self.timer = 5.0
        for _ in range(100):
            self.particles.emit(random.randint(0, SCREEN_WIDTH), random.randint(0, SCREEN_HEIGHT), COLORS['VICTORY'], 1)

    def update(self, dt):
        if not self.active:
            return False
        self.timer -= dt
        self.particles.update(dt)
        if self.timer <= 0:
            self.active = False
            return True
        return False

    def draw(self):
        if not self.active:
            return
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT))
        overlay.set_alpha(200)
        overlay.fill(COLORS['BG'])
        self.screen.blit(overlay, (0, 0))
        victory_text = self.font_big.render("ПОБЕДА!", True, COLORS['VICTORY'])
        text_rect = victory_text.get_rect(center=(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2 - 50))
        self.screen.blit(victory_text, text_rect)

class DefeatScreen:
    def __init__(self, screen):
        self.screen = screen
        self.active = False
        self.timer = 0
        self.font_big = pygame.font.Font(None, 80)
        self.font_medium = pygame.font.Font(None, 48)
        self.font_small = pygame.font.Font(None, 32)
        self.particles = ParticleSystem()

    def start(self):
        self.active = True
        self.timer = 5.0
        for _ in range(150):
            self.particles.emit(random.randint(0, SCREEN_WIDTH), random.randint(0, SCREEN_HEIGHT), COLORS['DEFEAT'], 1)

    def update(self, dt):
        if not self.active:
            return False
        self.timer -= dt
        self.particles.update(dt)
        if self.timer <= 0:
            self.active = False
            return True
        return False

    def draw(self):
        if not self.active:
            return
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT))
        overlay.set_alpha(200)
        overlay.fill(COLORS['BG'])
        self.screen.blit(overlay, (0, 0))
        defeat_text = self.font_big.render("ПОРАЖЕНИЕ", True, COLORS['DEFEAT'])
        text_rect = defeat_text.get_rect(center=(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2 - 80))
        self.screen.blit(defeat_text, text_rect)
        message_text = self.font_medium.render("Вы пали в бою...", True, COLORS['TEXT'])
        message_rect = message_text.get_rect(center=(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2))
        self.screen.blit(message_text, message_rect)
```
## 1.11 Главный класс - Game
```python
class Game:
    def __init__(self):
        self.screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
        pygame.display.set_caption("Изгнание нежити")
        self.clock = pygame.time.Clock()
        self.running = True
        self.dt = 0
        self.victory = False
        self.defeat = False

        self.background = self.load_background()
        self.player = Player(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2)
        self.enemies = self.create_enemies()
        self.npcs = self.create_npcs()
        self.items = self.create_items()

        self.particles = ParticleSystem()
        self.combat = None
        self.dialogue = None
        self.victory_screen = VictoryScreen(self.screen)
        self.defeat_screen = DefeatScreen(self.screen)

        self.show_inventory = False
        self.interact_cooldown = 0

    def create_enemies(self):
        return [
            Enemy(300, 200, 'skeleton', 2),
            Enemy(750, 410, 'skeleton', 2),
            Enemy(670, 350, 'skeleton', 2),
            Enemy(300, 300, 'ghoul', 2),
            Enemy(900, 350, 'lich', 1),
        ]

    def create_npcs(self):
        return [
            NPC(70, 190, 'merchant'),
            NPC(100, 300, 'elder'),
            NPC(50, 379, 'healer'),
        ]

    def create_items(self):
        return [
            Item(350, 400, 'potion'),
            Item(650, 200, 'gold'),
            Item(250, 400, 'potion'),
            Item(550, 350, 'gold'),
        ]

    def handle_events(self):
        for event in pygame.event.get():
            if event.type == QUIT:
                self.running = False
            if self.dialogue:
                self.dialogue.handle_event(event)
                if not self.dialogue.active:
                    self.dialogue = None
                continue
            if self.combat:
                self.combat.handle_event(event)
                continue
            if event.type == KEYDOWN:
                if event.key == K_ESCAPE:
                    self.running = False
                elif event.key == K_i:
                    self.show_inventory = not self.show_inventory
                elif event.key == K_e and self.interact_cooldown <= 0:
                    self.interact()

    def interact(self):
        for npc in self.npcs:
            if self.player.rect.colliderect(npc.rect):
                if npc.id == 'merchant' and self.interact_cooldown <= 0:
                    if self.player.gold >= 30:
                        self.player.gold -= 30
                        self.player.potions += 1
                        self.dialogue = DialogueBox(self.screen)
                        self.dialogue.start(["Вы купили зелье здоровья!", "Спасибо за покупку!"])
                        self.interact_cooldown = 0.5
                elif npc.id == 'healer' and self.interact_cooldown <= 0:
                    if self.player.gold >= 20 and self.player.hp < self.player.max_hp:
                        self.player.gold -= 20
                        self.player.hp = self.player.max_hp
                        self.dialogue = DialogueBox(self.screen)
                        self.dialogue.start(["Вы восстановили здоровье!", "Будь осторожен!"])
                        self.interact_cooldown = 0.5
                elif npc.id == 'elder':
                    self.dialogue = DialogueBox(self.screen)
                    self.dialogue.start(npc.dialogues)
                    self.interact_cooldown = 0.5
                return

    def update(self):
        if self.interact_cooldown > 0:
            self.interact_cooldown -= self.dt

        if self.victory:
            if self.victory_screen.update(self.dt):
                self.running = False
            return
        if self.defeat:
            if self.defeat_screen.update(self.dt):
                self.running = False
            return

        if self.combat is None and self.dialogue is None and not self.show_inventory:
            keys = pygame.key.get_pressed()
            dx = dy = 0
            if keys[K_LEFT] or keys[K_a]: dx = -1
            if keys[K_RIGHT] or keys[K_d]: dx = 1
            if keys[K_UP] or keys[K_w]: dy = -1
            if keys[K_DOWN] or keys[K_s]: dy = 1
            if dx != 0 or dy != 0:
                length = math.sqrt(dx*dx + dy*dy)
                dx /= length
                dy /= length
            new_x = self.player.x + dx * self.player.move_speed * self.dt
            new_y = self.player.y + dy * self.player.move_speed * self.dt
            if 0 <= new_x <= SCREEN_WIDTH - SPRITE_SIZE_MAP and 0 <= new_y <= SCREEN_HEIGHT - SPRITE_SIZE_MAP:
                self.player.x = new_x
                self.player.y = new_y
                self.player.rect.x = self.player.x
                self.player.rect.y = self.player.y

        self.particles.update(self.dt)

        if self.combat is None:
            for enemy in self.enemies[:]:
                if self.player.rect.colliderect(enemy.rect):
                    self.combat = CombatSystem(self.screen, self.player, enemy, self.particles)
                    break

        if self.combat:
            combat_end = self.combat.update(self.dt)
            if combat_end:
                if not self.player.is_alive():
                    self.defeat = True
                    self.defeat_screen.start()
                elif not self.combat.enemy.is_alive():
                    self.player.gain_exp(self.combat.enemy.exp_reward)
                    self.player.gold += self.combat.enemy.gold_reward
                    if self.combat.enemy in self.enemies:
                        self.enemies.remove(self.combat.enemy)
                self.combat = None

        if self.dialogue:
            self.dialogue.update(self.dt)

        for item in self.items[:]:
            if self.player.rect.colliderect(item.rect):
                if item.type == 'potion':
                    self.player.potions += 1
                elif item.type == 'gold':
                    self.player.gold += item.value
                elif item.type == 'sword':
                    self.player.equipment['weapon'] = {'name': 'Стальной меч', 'damage': 12}
                self.items.remove(item)

    def run(self):
        while self.running:
            self.dt = self.clock.tick(FPS) / 1000.0
            self.handle_events()
            self.update()
            self.draw()
        pygame.quit()
        sys.exit()
```
## 2 Заключение
Разработанная 2D RPG "Изгнание нежити" включает:

Анимированного персонажа с системой уровней и маны

Трёх типов врагов (Скелет, Мертвец, Лич)

NPC с диалогами, торговлей и лечением

Систему предметов (зелья, золото, оружие)

Боевую систему с выбором действий (атака, сильный удар, зелье)

Эффекты частиц при ударах и лечении

Экраны победы и поражения

Систему сохранения и загрузки (F5 – сохранить, F9 – загрузить)

Управление игрой:

WASD / Стрелки – движение

E – взаимодействие с NPC

I – инвентарь

F5 – сохранить игру

F9 – загрузить игру

ESC – выход

В бою:

↑ / ↓ – выбор действия

ENTER / ПРОБЕЛ – подтверждение

Проект демонстрирует все ключевые аспекты создания 2D RPG на Python с использованием Pygame и может служить основой для дальнейшего развития.
