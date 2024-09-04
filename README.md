import pygame
import random
import os

# Initialize Pygame
pygame.init()

# Screen dimensions
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Shooting Spaceship")

# Load and resize images
def load_and_resize_image(image_path, width, height):
    image = pygame.image.load(image_path)
    return pygame.transform.scale(image, (width, height))

background = load_and_resize_image('background.jpeg', SCREEN_WIDTH, SCREEN_HEIGHT)
spaceship_img = load_and_resize_image('spaceship.jpeg', 50, 50)
bullet_img = load_and_resize_image('bullet.jpeg', 20, 10)
monster_img = load_and_resize_image('monster.jpeg', 50, 50)

# Colors
WHITE = (255, 255, 255)
RED = (255, 0, 0)
GREEN = (0, 255, 0)

# Game variables
spaceship_width = 50
spaceship_height = 50
bullet_width = 20
bullet_height = 10
monster_width = 50
monster_height = 50

spaceship_x = SCREEN_WIDTH // 2 - spaceship_width // 2
spaceship_y = SCREEN_HEIGHT - spaceship_height - 10

bullet_speed = 10
monster_speed = 10
monster_spawn_interval = 1500  # Milliseconds
bullet_damage = 1
monster_health = 2
spaceship_health = 6
score = 0
best_score = 0  # Initialize best_score
font = pygame.font.SysFont(None, 36)

# File to store best score
score_file = 'best_score.txt'

# Load best score from file
def load_best_score():
    global best_score
    if os.path.exists(score_file):
        with open(score_file, 'r') as file:
            try:
                best_score = int(file.read())
            except ValueError:
                best_score = 0

# Save best score to file
def save_best_score():
    global best_score
    with open(score_file, 'w') as file:
        file.write(str(best_score))

# Lists to store bullets and monsters
bullets = []
monsters = []
game_over = False

# Timer for spawning monsters
pygame.time.set_timer(pygame.USEREVENT + 1, monster_spawn_interval)

# Bullet firing control
bullet_fire_rate = 200  # Milliseconds
last_bullet_time = 0

# Invincibility period after getting hit (in milliseconds)
invincibility_period = 1000
last_hit_time = 0

def draw_text(text, font, color, surface, x, y):
    textobj = font.render(text, True, color)
    textrect = textobj.get_rect()
    textrect.center = (x, y)
    surface.blit(textobj, textrect)

def draw_health_bar(surface, x, y, width, height, health, max_health):
    ratio = health / max_health
    fill = width * ratio
    pygame.draw.rect(surface, RED, (x, y, width, height))
    pygame.draw.rect(surface, GREEN, (x, y, fill, height))

def reset_game():
    global bullets, monsters, score, game_over, spaceship_health
    bullets = []
    monsters = []
    score = 0
    game_over = False
    spaceship_health = 6

def main():
    global spaceship_x, game_over, score, last_bullet_time, spaceship_health, best_score, last_hit_time

    load_best_score()  # Load best score at the start
    clock = pygame.time.Clock()
    reset_game()
    
    while True:
        current_time = pygame.time.get_ticks()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                save_best_score()  # Save best score when quitting
                pygame.quit()
                return
            
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_SPACE:
                    if game_over:
                        reset_game()
                    else:
                        # Check if enough time has passed to fire a new bullet
                        if current_time - last_bullet_time >= bullet_fire_rate:
                            bullet_rect = pygame.Rect(spaceship_x + spaceship_width // 2 - bullet_width // 2, spaceship_y, bullet_width, bullet_height)
                            bullets.append(bullet_rect)
                            last_bullet_time = current_time
                
            if event.type == pygame.USEREVENT + 1:
                if not game_over:
                    x = random.randint(0, SCREEN_WIDTH - monster_width)
                    y = -monster_height
                    monsters.append({'rect': pygame.Rect(x, y, monster_width, monster_height), 'health': monster_health})

        if not game_over:
            keys = pygame.key.get_pressed()
            if keys[pygame.K_LEFT] and spaceship_x > 0:
                spaceship_x -= 15
            if keys[pygame.K_RIGHT] and spaceship_x < SCREEN_WIDTH - spaceship_width:
                spaceship_x += 15
        
            # Move bullets
            for bullet in bullets[:]:
                bullet.y -= bullet_speed
                if bullet.y < 0:
                    bullets.remove(bullet)
            
            # Move monsters
            for monster in monsters[:]:
                monster['rect'].y += monster_speed
                if monster['rect'].y > SCREEN_HEIGHT:
                    monsters.remove(monster)
                    # Reduce spaceship health when monsters pass by
                    if current_time - last_hit_time > invincibility_period:
                        spaceship_health -= 1
                        last_hit_time = current_time
                        if spaceship_health <= 0:
                            game_over = True
                elif pygame.Rect(spaceship_x, spaceship_y, spaceship_width, spaceship_height).colliderect(monster['rect']):
                    if current_time - last_hit_time > invincibility_period:
                        spaceship_health -= 1
                        last_hit_time = current_time
                        if spaceship_health <= 0:
                            game_over = True
                        monsters.remove(monster)
            
            # Check bullet collisions
            for bullet in bullets[:]:
                for monster in monsters[:]:
                    if bullet.colliderect(monster['rect']):
                        bullets.remove(bullet)
                        monster['health'] -= bullet_damage
                        if monster['health'] <= 0:
                            monsters.remove(monster)
                            score += 1
                        break

        # Update best score
        if score > best_score:
            best_score = score

        screen.blit(background, (0, 0))
        
        # Draw spaceship
        screen.blit(spaceship_img, (spaceship_x, spaceship_y))
        draw_health_bar(screen, spaceship_x, spaceship_y - 20, spaceship_width, 10, spaceship_health, 6)
        
        # Draw bullets
        for bullet in bullets:
            screen.blit(bullet_img, (bullet.x, bullet.y))
        
        # Draw monsters
        for monster in monsters:
            screen.blit(monster_img, (monster['rect'].x, monster['rect'].y))
            draw_health_bar(screen, monster['rect'].x, monster['rect'].y - 20, monster_width, 10, monster['health'], monster_health)
        
        # Draw score and best score
        draw_text(f"Score: {score}", font, WHITE, screen, SCREEN_WIDTH // 2, 30)
        draw_text(f"Best Score: {best_score}", font, WHITE, screen, SCREEN_WIDTH // 2, 70)
        
        if game_over:
            draw_text("Game Over! Press SPACE to Restart", font, WHITE, screen, SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2)
        
        pygame.display.flip()
        clock.tick(30)

if __name__ == "__main__":
    main()
