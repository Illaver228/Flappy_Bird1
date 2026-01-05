import pygame
from pygame import Rect, draw
from random import randint
import pyaudio
import numpy as np
import threading
import sys
import time

pygame.init()

# ===== WINDOW =====
window_size = (800, 600)
window = pygame.display.set_mode(window_size)
pygame.display.set_caption("Flappy Bird (Voice Control)")
clock = pygame.time.Clock()

# ===== PLAYER =====
player_rect = Rect(80, 250, 60, 60)
velocity = 0
gravity = 1.2
lift = -8

# ===== MICROPHONE =====
CHUNK = 1024
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100

raw_volume = 0
smooth_volume = 0

SMOOTHING = 0.9
VOLUME_THRESHOLD = 6000

def listen_volume():
    global raw_volume
    p = pyaudio.PyAudio()
    stream = p.open(
        format=FORMAT,
        channels=CHANNELS,
        rate=RATE,
        input=True,
        frames_per_buffer=CHUNK
    )

    while True:
        data = stream.read(CHUNK, exception_on_overflow=False)
        audio = np.frombuffer(data, dtype=np.int16)
        raw_volume = np.linalg.norm(audio)

threading.Thread(target=listen_volume, daemon=True).start()

# ===== PIPES =====
def generate_pipes(count, pipe_width, gap=200, min_height=50, max_height=350, distance=450):
    pipes = []
    start_x = window_size[0]
    for _ in range(count):
        h = randint(min_height, max_height)
        pipes.append(Rect(start_x, 0, pipe_width, h))
        pipes.append(Rect(start_x, h + gap, pipe_width, window_size[1] - h - gap))
        start_x += distance
    return pipes

# ===== RESET GAME =====
def reset_game():
    global pipes, player_rect, velocity
    player_rect.y = 250
    velocity = 0
    pipes = generate_pipes(3, 100)

reset_game()

# ===== FONTS =====
font = pygame.font.Font(None, 80)
small_font = pygame.font.Font(None, 28)

# ===== INDICATOR =====
BAR_X = 20
BAR_Y = 20
BAR_WIDTH = 200
BAR_HEIGHT = 20
MAX_VOLUME = 12000

# ===== GAME LOOP =====
while True:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()

    # ===== SMOOTH VOLUME =====
    smooth_volume = SMOOTHING * smooth_volume + (1 - SMOOTHING) * raw_volume

    # ===== VOICE CONTROL =====
    if smooth_volume > VOLUME_THRESHOLD:
        velocity = lift
    else:
        velocity += gravity

    player_rect.y += int(velocity)

    # ===== LIMITS =====
    if player_rect.top < 0:
        player_rect.top = 0
        velocity = 0
    if player_rect.bottom > window_size[1]:
        player_rect.bottom = window_size[1]
        velocity = 0

    # ===== DRAW =====
    window.fill("skyblue")
    draw.rect(window, "yellow", player_rect)

    # ===== PIPES =====
    for pipe in pipes[:]:
        pipe.x -= 6
        draw.rect(window, "green", pipe)
        if pipe.x < -120:
            pipes.remove(pipe)

    if len(pipes) < 6:
        pipes += generate_pipes(2, 100)

    # ===== COLLISION =====
    for pipe in pipes:
        if player_rect.colliderect(pipe):
            text = font.render("CRASH!", True, "red")
            window.blit(
                text,
                (
                    window_size[0] // 2 - text.get_width() // 2,
                    window_size[1] // 2 - text.get_height() // 2
                )
            )
            pygame.display.update()
            pygame.time.delay(1000)
            reset_game()
            break

    # ===== VOLUME INDICATOR =====
    draw.rect(window, "black", (BAR_X - 2, BAR_Y - 2, BAR_WIDTH + 4, BAR_HEIGHT + 4), 2)

    volume_ratio = min(smooth_volume / MAX_VOLUME, 1)
    draw.rect(
        window,
        "green",
        (BAR_X, BAR_Y, int(BAR_WIDTH * volume_ratio), BAR_HEIGHT)
    )

    threshold_x = BAR_X + int(BAR_WIDTH * (VOLUME_THRESHOLD / MAX_VOLUME))
    draw.line(
        window,
        "red",
        (threshold_x, BAR_Y),
        (threshold_x, BAR_Y + BAR_HEIGHT),
        3
    )

    label = small_font.render("Voice volume", True, "black")
    window.blit(label, (BAR_X, BAR_Y + 25))

    pygame.display.update()
    clock.tick(60)
