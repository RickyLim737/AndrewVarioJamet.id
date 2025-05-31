import pygame
import sys
import sounddevice as sd
import numpy as np
import threading
import random
import time

WIDTH, HEIGHT = 800, 400
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
GROUND_Y = 350
SOUND_THRESHOLD_DB = 5
MAX_OBSTACLES = 2
SPAWN_DELAY = 1500

sound_volume_db = 0
sound_lock = threading.Lock()

import time

def listen_for_sound():
    global sound_volume_db
    mic_index = None
    try:
        devices = sd.query_devices()
        for i, dev in enumerate(devices):
            if dev['max_input_channels'] > 0:
                mic_index = i
                break
        if mic_index is None:
            print("No input device found.")
            return
    except Exception as e:
        print("Error querying devices:", e)
        return

    while True:
        try:
            duration = 0.1
            audio = sd.rec(int(duration * 44100), samplerate=44100, channels=1, dtype='float64', device=mic_index)
            sd.wait()
            amplitude = np.max(np.abs(audio))

            with sound_lock:
                if amplitude > 0:
                    sound_volume_db = 20 * np.log10(amplitude)
                else:
                    sound_volume_db = -np.inf

            time.sleep(0.05)
        except Exception as e:
            print("Error reading sound:", e)

class Obstacle:
    def __init__(self, x, speed):
        self.rect = pygame.Rect(x, GROUND_Y - 30, 20, 30)
        self.base_speed = speed
        self.has_scored = False

    def update(self):
        self.rect.x -= self.base_speed

    def draw(self, screen):
        pygame.draw.rect(screen, (0, 128, 0), self.rect)

    def off_screen(self):
        return self.rect.right < 0

class Player:
    def __init__(self):
        self.rect = pygame.Rect(100, GROUND_Y - 50, 50, 50)
        try:
            self.image = pygame.image.load("player.png").convert_alpha()
            self.image = pygame.transform.scale(self.image, (50, 50))
        except:
            self.image = None
        self.gravity = 0
        self.is_jumping = False

    def update(self):
        if self.is_jumping:
            self.gravity += 1
            self.rect.y += self.gravity
            print(f"Player Y Position: {self.rect.y}")
            if self.rect.y >= GROUND_Y - 50:
                self.rect.y = GROUND_Y - 50
                self.is_jumping = False
                self.gravity = 0


    def jump(self, power=1.0):
        if not self.is_jumping:
            self.is_jumping = True
            self.gravity = -int(10 + 10 * power)

    def draw(self, screen):
        if self.image:
            screen.blit(self.image, self.rect)
        else:
            pygame.draw.rect(screen, (255, 0, 0), self.rect)

class Game:
    def __init__(self):
        pygame.init()
        pygame.font.init()
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        pygame.display.set_caption("Sound-Controlled Dino Game")
        self.clock = pygame.time.Clock()
        self.font = pygame.font.Font(None, 48)
        self.small_font = pygame.font.Font(None, 36)

        self.in_menu = True
        self.show_highscore = False
        self.game_over = False
        self.choose_input_method = False
        self.use_sound = False
        self.use_keyboard = False

        self.score = 0
        self.high_score = 0
        self.player = Player()
        self.obstacles = []
        self.last_spawn_time = pygame.time.get_ticks()
        self.target_obstacle_count = random.randint(1, MAX_OBSTACLES)
        self.last_target_change_time = pygame.time.get_ticks()
        self.target_change_interval = 5000

        self.play_button = pygame.Rect(WIDTH // 2 - 100, HEIGHT // 2 - 100, 200, 50)
        self.highscore_button = pygame.Rect(WIDTH // 2 - 100, HEIGHT // 2 - 30, 200, 50)
        self.exit_button = pygame.Rect(WIDTH // 2 - 100, HEIGHT // 2 + 40, 200, 50)

        self.voice_button_text = "Voice Control"
        self.keyboard_button_text = "Keyboard"
        self.voice_button = None
        self.keyboard_button = None

        self.restart_button = pygame.Rect(WIDTH // 2 - 100, HEIGHT // 2 + 40, 200, 50)
        self.back_to_menu_button = pygame.Rect(WIDTH // 2 - 100, HEIGHT // 2 + 110, 200, 50)

    def run(self):
        global sound_volume_db
        while True:
            self.handle_events()

            if self.in_menu:
                if self.show_highscore:
                    self.draw_highscore_menu()
                elif self.choose_input_method:
                    self.draw_choose_input_menu()
                else:
                    self.draw_menu()
            elif self.game_over:
                self.draw_game_over_menu()
            else:
                self.update()
                self.draw()
                self.clock.tick(60)

                if self.use_sound:
                    with sound_lock:
                        vol = sound_volume_db
                    print(f"Volume: {vol}")
                    if vol > SOUND_THRESHOLD_DB:
                        jump_power = min((vol - SOUND_THRESHOLD_DB) / 0.5, 1.5)
                        print(f"Jump Power: {jump_power}")
                        self.player.jump(jump_power)



    def reset_game(self):
        self.player = Player()
        self.obstacles = []
        self.last_spawn_time = pygame.time.get_ticks()
        self.target_obstacle_count = random.randint(1, MAX_OBSTACLES)
        self.last_target_change_time = pygame.time.get_ticks()
        self.score = 0
        self.game_over = False

    def handle_events(self):
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

            if self.in_menu:
                if self.show_highscore:
                    if event.type == pygame.MOUSEBUTTONDOWN:
                        self.show_highscore = False
                elif self.choose_input_method:
                    if event.type == pygame.MOUSEBUTTONDOWN:
                        mouse_pos = pygame.mouse.get_pos()
                        if self.voice_button and self.voice_button.collidepoint(mouse_pos):
                            self.use_sound = True
                            self.use_keyboard = False
                            self.in_menu = False
                            self.choose_input_method = False
                            self.reset_game()
                        elif self.keyboard_button and self.keyboard_button.collidepoint(mouse_pos):
                            self.use_keyboard = True
                            self.use_sound = False
                            self.in_menu = False
                            self.choose_input_method = False
                            self.reset_game()
                else:
                    if event.type == pygame.MOUSEBUTTONDOWN:
                        mouse_pos = pygame.mouse.get_pos()
                        if self.play_button.collidepoint(mouse_pos):
                            self.choose_input_method = True
                        elif self.exit_button.collidepoint(mouse_pos):
                            pygame.quit()
                            sys.exit()
                        elif self.highscore_button.collidepoint(mouse_pos):
                            self.show_highscore = True

            elif self.game_over:
                if event.type == pygame.MOUSEBUTTONDOWN:
                    mouse_pos = pygame.mouse.get_pos()
                    if self.restart_button.collidepoint(mouse_pos):
                        self.reset_game()
                        self.in_menu = False
                    elif self.back_to_menu_button.collidepoint(mouse_pos):
                        self.in_menu = True
                        self.game_over = False
            else:
                if self.use_keyboard and event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_SPACE:
                        self.player.jump()

    def update(self):
        self.player.update()
        now = pygame.time.get_ticks()

        speed_bonus = min(self.score // 2, 5)
        obstacle_speed = 5 + speed_bonus

        for obs in self.obstacles:
            if self.player.rect.colliderect(obs.rect):
                self.game_over = True
                if self.score > self.high_score:
                    self.high_score = self.score
                return

        for obs in self.obstacles:
            if not obs.has_scored and self.player.rect.left > obs.rect.right:
                self.score += 1
                obs.has_scored = True

        if now - self.last_target_change_time > self.target_change_interval:
            self.target_obstacle_count = random.randint(1, MAX_OBSTACLES)
            self.last_target_change_time = now

        if len(self.obstacles) < self.target_obstacle_count:
            if now - self.last_spawn_time > SPAWN_DELAY:
                can_spawn = self.target_obstacle_count - len(self.obstacles)
                block_size = random.randint(1, can_spawn)
                start_x = WIDTH + random.randint(50, 150)

                for i in range(block_size):
                    if block_size == 1:
                        y = GROUND_Y - 30
                        obs = Obstacle(start_x, obstacle_speed)
                    else:
                        y = GROUND_Y - 30 - (35 * i)
                        obs = Obstacle(start_x, obstacle_speed)
                    obs.rect.y = y
                    self.obstacles.append(obs)

                self.last_spawn_time = now

        for obs in self.obstacles:
            obs.base_speed = obstacle_speed
            obs.update()

        self.obstacles = [obs for obs in self.obstacles if not obs.off_screen()]

    def draw_text_centered(self, text, font, color, rect):
        text_surface = font.render(text, True, color)
        text_rect = text_surface.get_rect(center=rect.center)
        self.screen.blit(text_surface, text_rect)

    def draw_menu(self):
        self.screen.fill(WHITE)
        title = self.font.render("Sound Dino Game", True, BLACK)
        self.screen.blit(title, (WIDTH // 2 - title.get_width() // 2, 40))

        pygame.draw.rect(self.screen, (0, 200, 0), self.play_button)
        pygame.draw.rect(self.screen, (0, 0, 200), self.highscore_button)
        pygame.draw.rect(self.screen, (200, 0, 0), self.exit_button)

        self.draw_text_centered("Play", self.font, WHITE, self.play_button)
        self.draw_text_centered("High Score", self.font, WHITE, self.highscore_button)
        self.draw_text_centered("Exit", self.font, WHITE, self.exit_button)

        pygame.display.flip()

    def draw_choose_input_menu(self):
        self.screen.fill(WHITE)
        title = self.font.render("Choose Input Method", True, BLACK)
        self.screen.blit(title, (WIDTH // 2 - title.get_width() // 2, 40))

        button_width = 250
        button_height = 70
        gap = 40

        total_width = button_width * 2 + gap
        start_x = (WIDTH - total_width) // 2
        y_pos = HEIGHT // 2 - button_height // 2

        voice_button_rect = pygame.Rect(start_x, y_pos, button_width, button_height)
        keyboard_button_rect = pygame.Rect(start_x + button_width + gap, y_pos, button_width, button_height)

        self.voice_button = voice_button_rect
        self.keyboard_button = keyboard_button_rect

        pygame.draw.rect(self.screen, (0, 150, 0), voice_button_rect, border_radius=8)
        pygame.draw.rect(self.screen, (0, 0, 150), keyboard_button_rect, border_radius=8)

        voice_text_surf = self.font.render(self.voice_button_text, True, WHITE)
        keyboard_text_surf = self.font.render(self.keyboard_button_text, True, WHITE)

        voice_text_rect = voice_text_surf.get_rect(center=voice_button_rect.center)
        keyboard_text_rect = keyboard_text_surf.get_rect(center=keyboard_button_rect.center)

        self.screen.blit(voice_text_surf, voice_text_rect)
        self.screen.blit(keyboard_text_surf, keyboard_text_rect)

        pygame.display.flip()

    def draw_highscore_menu(self):
        self.screen.fill(WHITE)
        title = self.font.render("High Score", True, BLACK)
        self.screen.blit(title, (WIDTH // 2 - title.get_width() // 2, 100))

        score_text = self.font.render(f"Highest Score: {self.high_score}", True, BLACK)
        self.screen.blit(score_text, (WIDTH // 2 - score_text.get_width() // 2, 200))

        info_text = self.small_font.render("Click anywhere to return", True, (100, 100, 100))
        self.screen.blit(info_text, (WIDTH // 2 - info_text.get_width() // 2, 300))

        pygame.display.flip()

    def draw_game_over_menu(self):
        self.screen.fill(WHITE)
        game_over_text = self.font.render("Game Over!", True, (255, 0, 0))
        self.screen.blit(game_over_text, (WIDTH // 2 - game_over_text.get_width() // 2, 80))

        score_text = self.font.render(f"Score: {self.score}", True, BLACK)
        self.screen.blit(score_text, (WIDTH // 2 - score_text.get_width() // 2, 80 + game_over_text.get_height() + 20))

        pygame.draw.rect(self.screen, (0, 150, 255), self.restart_button)
        pygame.draw.rect(self.screen, (100, 100, 100), self.back_to_menu_button)

        self.draw_text_centered("Restart", self.font, WHITE, self.restart_button)
        self.draw_text_centered("Main Menu", self.font, WHITE, self.back_to_menu_button)

        pygame.display.flip()

    def draw(self):
        self.screen.fill(WHITE)
        pygame.draw.line(self.screen, BLACK, (0, GROUND_Y), (WIDTH, GROUND_Y), 2)
        self.player.draw(self.screen)
        for obs in self.obstacles:
            obs.draw(self.screen)

        score_text = self.small_font.render(f"Score: {self.score}", True, BLACK)
        self.screen.blit(score_text, (10, 10))

        pygame.display.flip()

if __name__ == "__main__":
    print("List of input devices:")
    print(sd.query_devices())
    print("Default input device:", sd.default.device)
    threading.Thread(target=listen_for_sound, daemon=True).start()
    Game().run()
