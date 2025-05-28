import pygame
import random
import math

from pygments.styles.dracula import background

pygame.init()

screen_width, screen_height = 1000, 700
screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption("Zombie Survival")
clock = pygame.time.Clock()

background_color = (15, 15, 25)
player_color = (255, 100, 100)
zombie_color = (0, 200, 0)
special_zombie_color = (255, 0, 0)
blast_color = (255, 255, 100)
blastometer_bg_color = (50, 50, 50)
blastometer_fill_color = (255, 215, 0)

player_radius = 30
player_pos_x = screen_width // 2
player_pos_y = screen_height // 2

zombie_block_size = 30
zombie_list = []

game_start_time = pygame.time.get_ticks()
last_zombie_spawn_time = 0

blast_active = False
blast_start_time = 0
blast_current_radius = 0

current_level = 1
zombie_move_speed = 1.5
chance_special_zombie = 0.1

blastometer_max_value = 100
blastometer_value = 0
blastometer_fill_per_ms = blastometer_max_value / 25000  # fills fully in 25 seconds

def draw_player(x, y):
    pygame.draw.circle(screen, player_color, (int(x), int(y)), player_radius)

def draw_zombie(zombie):
    zx, zy = zombie['position']
    color = special_zombie_color if zombie['special'] else zombie_color
    rect = pygame.Rect(int(zx - zombie_block_size/2), int(zy - zombie_block_size/2), zombie_block_size, zombie_block_size)
    pygame.draw.rect(screen, color, rect)

def spawn_new_zombie():
    side = random.choice(['top', 'bottom', 'left', 'right'])
    if side == 'top':
        x, y = random.randint(0, screen_width), -zombie_block_size
    elif side == 'bottom':
        x, y = random.randint(0, screen_width), screen_height + zombie_block_size
    elif side == 'left':
        x, y = -zombie_block_size, random.randint(0, screen_height)
    else:
        x, y = screen_width + zombie_block_size, random.randint(0, screen_height)
    special = random.random() < chance_special_zombie
    zombie_list.append({'position': [x, y], 'special': special})

def update_zombie_positions():
    for zombie in zombie_list:
        zx, zy = zombie['position']
        dx, dy = player_pos_x - zx, player_pos_y - zy
        distance = math.hypot(dx, dy)
        if distance != 0:
            zombie['position'][0] += zombie_move_speed * dx / distance
            zombie['position'][1] += zombie_move_speed * dy / distance

def player_collided():
    for zombie in zombie_list:
        zx, zy = zombie['position']
        distance = math.hypot(player_pos_x - zx, player_pos_y - zy)
        if distance < (player_radius + zombie_block_size) / 2:
            return True
    return False

def activate_blast():
    global blast_active, blast_start_time, blast_current_radius, blastometer_value
    blast_active = True
    blast_start_time = pygame.time.get_ticks()
    blast_current_radius = 50
    blastometer_value = 0

def draw_light_around_player():
    mask_surface = pygame.Surface((screen_width, screen_height))
    mask_surface.fill(background_color)
    pygame.draw.circle(mask_surface, (0, 0, 0), (int(player_pos_x), int(player_pos_y)), 150)
    mask_surface.set_colorkey((0, 0, 0))
    mask_surface.set_alpha(230)
    screen.blit(mask_surface, (0, 0))

def draw_blastometer_bar():
    bar_width, bar_height = 200, 25
    bar_x, bar_y = screen_width - bar_width - 20, screen_height - bar_height - 20
    pygame.draw.rect(screen, blastometer_bg_color, (bar_x, bar_y, bar_width, bar_height), border_radius=5)
    filled_width = (blastometer_value / blastometer_max_value) * bar_width
    pygame.draw.rect(screen, blastometer_fill_color, (bar_x, bar_y, filled_width, bar_height), border_radius=5)
    pygame.draw.rect(screen, (255, 255, 255), (bar_x, bar_y, bar_width, bar_height), 2, border_radius=5)
    font = pygame.font.SysFont(None, 40)
    label = font.render("blastometer", True, (255, 255, 255))
    screen.blit(label, (bar_x, bar_y - 30))

def display_text(message, vertical_pos, font_size=40, color=(255, 255, 255), center_aligned=True):
    font = pygame.font.SysFont(None, font_size)
    text_surface = font.render(message, True, color)
    if center_aligned:
        text_rect = text_surface.get_rect(center=(screen_width // 2, vertical_pos))
    else:
        text_rect = text_surface.get_rect(topleft=(20, vertical_pos))
    screen.blit(text_surface, text_rect)

running = True
game_over = False
while running:
    delta_time = clock.tick(60)
    screen.fill(background_color)
    keys = pygame.key.get_pressed()

    for event in pygame.event.get():
        if event.type == pygame.QUIT or keys[pygame.K_q]:
            running = False

    if not game_over:
        mouse_x, mouse_y = pygame.mouse.get_pos()
        player_pos_x += (mouse_x - player_pos_x) * 0.1
        player_pos_y += (mouse_y - player_pos_y) * 0.1

        now = pygame.time.get_ticks()
        elapsed_seconds = (now - game_start_time) // 1000

        current_level = elapsed_seconds // 30 + 1
        spawn_interval = max(200, 2000 - (current_level - 1) * 250)
        zombie_move_speed = 1.5 + (current_level - 1) * 0.3
        chance_special_zombie = min(0.3, 0.1 + (current_level - 1) * 0.02)

        if now - last_zombie_spawn_time > spawn_interval:
            to_spawn = min(current_level, 5)
            for _ in range(to_spawn):
                spawn_new_zombie()
            last_zombie_spawn_time = now

        update_zombie_positions()

        if player_collided():
            game_over = True
            total_survival_time = elapsed_seconds

        for z in zombie_list:
            draw_zombie(z)

        draw_player(player_pos_x, player_pos_y)
        draw_light_around_player()

        if blastometer_value < blastometer_max_value:
            blastometer_value += blastometer_fill_per_ms * delta_time

        if keys[pygame.K_b] and blastometer_value >= blastometer_max_value and not blast_active:
            activate_blast()

        if blast_active:
            blast_current_radius += 15

            for z in zombie_list[:]:
                zx, zy = z['position']
                dist_to_player = math.hypot(player_pos_x - zx, player_pos_y - zy)
                if dist_to_player <= blast_current_radius:
                    zombie_list.remove(z)

            if blast_current_radius < 400:
                pygame.draw.circle(screen, blast_color, (int(player_pos_x), int(player_pos_y)), blast_current_radius, width=6)
            else:
                blast_active = False
                blast_current_radius = 0

        draw_blastometer_bar()

        display_text(f"survival time: {elapsed_seconds}s", 20)
        display_text(f"level: {current_level}", 60, font_size=30)

    else:
        display_text(" game over ", screen_height // 2 - 40, font_size=60, color=(255, 80, 80))
        display_text(f"you survived {total_survival_time} seconds", screen_height // 2 + 10)
        display_text("press r to restart or q to quit", screen_height // 2 + 60, font_size=30)
        if keys[pygame.K_r]:
            player_pos_x, player_pos_y = screen_width // 2, screen_height // 2
            zombie_list.clear()
            game_start_time = pygame.time.get_ticks()
            last_zombie_spawn_time = 0
            game_over = False
            blast_active = False
            blast_current_radius = 0
            blastometer_value = 0

    pygame.display.update()

pygame.quit()
