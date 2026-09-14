import tkinter as tk
from tkinter import messagebox
import random
import json
from pathlib import Path

CELL_SIZE = 25
COLS = 28
ROWS = 22
WIDTH = COLS * CELL_SIZE
HEIGHT = ROWS * CELL_SIZE

START_SPEED = 120
MIN_SPEED = 55
SPEED_STEP = 6

HIGH_SCORE_FILE = Path.home() / ".snake_game_highscore.json"

BG = "#111827"
GRID = "#1f2937"
SNAKE_HEAD = "#22c55e"
SNAKE_BODY = "#16a34a"
FOOD = "#ef4444"
TEXT = "#f8fafc"
OBSTACLE = "#64748b"
SPECIAL_FOOD = "#f59e0b"


class SnakeGame:

    def __init__(self, root):
        self.root = root
        self.root.title("Ultimate Snake Game")
        self.root.resizable(False, False)
        self.root.configure(bg=BG)

        self.score = 0
        self.level = 1
        self.speed = START_SPEED
        self.high_score = self.load_high_score()

        self.running = False
        self.paused = False
        self.game_over = False

        self.direction = "Right"
        self.next_direction = "Right"

        self.snake = []
        self.food = None
        self.special_food = None
        self.obstacles = set()

        self.special_food_timer = 0
        self.after_id = None

        self.build_ui()
        self.bind_keys()
        self.new_game()

    # ---------------- UI ----------------

    def build_ui(self):

        top = tk.Frame(self.root, bg=BG)
        top.pack(fill="x", padx=10, pady=(10, 5))

        self.score_label = tk.Label(
            top,
            text="",
            font=("Segoe UI", 12, "bold"),
            bg=BG,
            fg=TEXT
        )
        self.score_label.pack(side="left")

        self.level_label = tk.Label(
            top,
            text="",
            font=("Segoe UI", 12, "bold"),
            bg=BG,
            fg=TEXT
        )
        self.level_label.pack(side="left", padx=25)

        self.high_label = tk.Label(
            top,
            text="",
            font=("Segoe UI", 12, "bold"),
            bg=BG,
            fg=TEXT
        )
        self.high_label.pack(side="right")

        self.canvas = tk.Canvas(
            self.root,
            width=WIDTH,
            height=HEIGHT,
            bg=BG,
            highlightthickness=1,
            highlightbackground=GRID
        )
        self.canvas.pack(padx=10, pady=5)

        controls = tk.Frame(self.root, bg=BG)
        controls.pack(pady=(5, 10))

        tk.Button(
            controls,
            text="New Game",
            command=self.new_game,
            width=12
        ).pack(side="left", padx=4)

        self.pause_button = tk.Button(
            controls,
            text="Pause",
            command=self.toggle_pause,
            width=12
        )
        self.pause_button.pack(side="left", padx=4)

        tk.Label(
            self.root,
            text=(
                "Arrow Keys / WASD = Move   •   "
                "Space = Pause   •   R = Restart   •   Esc = Quit"
            ),
            font=("Segoe UI", 9),
            bg=BG,
            fg="#cbd5e1"
        ).pack(pady=(0, 10))

    # ---------------- KEYBOARD ----------------

    def bind_keys(self):

        self.root.bind("<KeyPress>", self.key_pressed)

        self.root.protocol(
            "WM_DELETE_WINDOW",
            self.close
        )

    def key_pressed(self, event):

        key = event.keysym.lower()

        directions = {
            "up": "Up",
            "w": "Up",

            "down": "Down",
            "s": "Down",

            "left": "Left",
            "a": "Left",

            "right": "Right",
            "d": "Right"
        }

        if key in directions:

            self.change_direction(
                directions[key]
            )

        elif key == "space":

            self.toggle_pause()

        elif key == "r":

            self.new_game()

        elif key == "escape":

            self.close()

    # ---------------- HIGH SCORE ----------------

    def load_high_score(self):

        try:

            if HIGH_SCORE_FILE.exists():

                data = json.loads(
                    HIGH_SCORE_FILE.read_text(
                        encoding="utf-8"
                    )
                )

                return max(
                    0,
                    int(data.get("high_score", 0))
                )

        except (
            OSError,
            ValueError,
            TypeError,
            json.JSONDecodeError
        ):
            pass

        return 0

    def save_high_score(self):

        try:

            HIGH_SCORE_FILE.write_text(
                json.dumps({
                    "high_score": self.high_score
                }),
                encoding="utf-8"
            )

        except OSError:
            pass

    # ---------------- NEW GAME ----------------

    def new_game(self):

        if self.after_id is not None:

            try:
                self.root.after_cancel(
                    self.after_id
                )

            except tk.TclError:
                pass

            self.after_id = None

        center_x = COLS // 2
        center_y = ROWS // 2

        self.snake = [
            (center_x, center_y),
            (center_x - 1, center_y),
            (center_x - 2, center_y),
            (center_x - 3, center_y)
        ]

        self.direction = "Right"
        self.next_direction = "Right"

        self.score = 0
        self.level = 1
        self.speed = START_SPEED

        self.running = True
        self.paused = False
        self.game_over = False

        self.special_food = None
        self.special_food_timer = 0

        self.obstacles = set()

        self.create_obstacles()

        self.food = self.random_empty_cell()

        self.update_ui()
        self.draw()

        self.game_loop()

    # ---------------- OBSTACLES ----------------

    def create_obstacles(self):

        count = min(
            3 + (self.level - 1) * 2,
            35
        )

        self.obstacles.clear()

        for _ in range(count):

            cell = self.random_empty_cell(
                extra_blocked=self.obstacles
            )

            if cell is not None:
                self.obstacles.add(cell)

    # ---------------- RANDOM CELL ----------------

    def random_empty_cell(self, extra_blocked=None):

        blocked = (
            set(self.snake)
            | self.obstacles
        )

        if extra_blocked:
            blocked |= set(extra_blocked)

        available = [
            (x, y)

            for y in range(ROWS)

            for x in range(COLS)

            if (x, y) not in blocked
        ]

        if not available:
            return None

        return random.choice(
            available
        )

    # ---------------- FOOD ----------------

    def spawn_food(self):

        self.food = self.random_empty_cell()

        if self.food is None:
            self.win_game()

    def maybe_spawn_special_food(self):

        if (
            self.special_food is None
            and self.score > 0
            and self.score % 5 == 0
        ):

            self.special_food = (
                self.random_empty_cell()
            )

            self.special_food_timer = 70

    # ---------------- DIRECTION ----------------

    def change_direction(self, new_direction):

        opposites = {
            "Up": "Down",
            "Down": "Up",
            "Left": "Right",
            "Right": "Left"
        }

        if new_direction != opposites.get(
            self.direction
        ):

            self.next_direction = new_direction

    # ---------------- PAUSE ----------------

    def toggle_pause(self):

        if (
            not self.running
            or self.game_over
        ):
            return

        self.paused = not self.paused

        self.pause_button.config(
            text=(
                "Resume"
                if self.paused
                else "Pause"
            )
        )

        self.draw()

        if not self.paused:
            self.game_loop()

    # ---------------- GAME LOOP ----------------

    def game_loop(self):

        if (
            not self.running
            or self.paused
            or self.game_over
        ):
            return

        self.move()

        if (
            self.running
            and not self.paused
            and not self.game_over
        ):

            self.after_id = self.root.after(
                self.speed,
                self.game_loop
            )

    # ---------------- MOVE ----------------

    def move(self):

        self.direction = self.next_direction

        head_x, head_y = self.snake[0]

        dx, dy = {
            "Up": (0, -1),
            "Down": (0, 1),
            "Left": (-1, 0),
            "Right": (1, 0)
        }[self.direction]

        new_head = (
            head_x + dx,
            head_y + dy
        )

        # Wall collision
        if not (
            0 <= new_head[0] < COLS
            and
            0 <= new_head[1] < ROWS
        ):

            self.end_game(
                "You hit the wall!"
            )

            return

        # Snake collision
        if new_head in self.snake:

            self.end_game(
                "You hit yourself!"
            )

            return

        # Obstacle collision
        if new_head in self.obstacles:

            self.end_game(
                "You hit an obstacle!"
            )

            return

        self.snake.insert(
            0,
            new_head
        )

        ate_food = (
            new_head == self.food
        )

        ate_special = (
            new_head == self.special_food
        )

        if ate_food:

            self.score += 1

            self.spawn_food()

            self.maybe_spawn_special_food()

            if self.score % 5 == 0:

                self.level += 1

                self.speed = max(
                    MIN_SPEED,
                    START_SPEED
                    -
                    (
                        self.level - 1
                    )
                    * SPEED_STEP
                )

                self.create_obstacles()

        elif ate_special:

            self.score += 5

            self.special_food = None

            self.special_food_timer = 0

            self.maybe_spawn_special_food()

        else:

            self.snake.pop()

        # Special food timer
        if self.special_food is not None:

            self.special_food_timer -= 1

            if self.special_food_timer <= 0:

                self.special_food = None

        # High score
        if self.score > self.high_score:

            self.high_score = self.score

            self.save_high_score()

        self.update_ui()
        self.draw()

    # ---------------- GAME OVER ----------------

    def end_game(self, reason):

        self.running = False
        self.game_over = True

        self.update_ui()
        self.draw()

        messagebox.showinfo(
            "Game Over",
            (
                f"{reason}\n\n"
                f"Score: {self.score}\n"
                f"Level: {self.level}\n"
                f"High Score: {self.high_score}\n\n"
                "Press R or click New Game "
                "to play again."
            )
        )

    # ---------------- WIN ----------------

    def win_game(self):

        self.running = False
        self.game_over = True

        self.draw()

        messagebox.showinfo(
            "You Win!",
            (
                "Congratulations!\n"
                "You filled the board.\n\n"
                f"Score: {self.score}"
            )
        )

    # ---------------- UI UPDATE ----------------

    def update_ui(self):

        self.score_label.config(
            text=f"Score: {self.score}"
        )

        self.level_label.config(
            text=f"Level: {self.level}"
        )

        self.high_label.config(
            text=f"High Score: {self.high_score}"
        )

    # ---------------- DRAWING ----------------

    def cell_rect(self, cell):

        x, y = cell

        return (
            x * CELL_SIZE,
            y * CELL_SIZE,
            (x + 1) * CELL_SIZE,
            (y + 1) * CELL_SIZE
        )

    def draw(self):

        self.canvas.delete("all")

        # Grid
        for x in range(
            0,
            WIDTH,
            CELL_SIZE
        ):

            self.canvas.create_line(
                x,
                0,
                x,
                HEIGHT,
                fill=GRID
            )

        for y in range(
            0,
            HEIGHT,
            CELL_SIZE
        ):

            self.canvas.create_line(
                0,
                y,
                WIDTH,
                y,
                fill=GRID
            )

        # Obstacles
        for obstacle in self.obstacles:

            x1, y1, x2, y2 = (
                self.cell_rect(obstacle)
            )

            self.canvas.create_rectangle(
                x1 + 2,
                y1 + 2,
                x2 - 2,
                y2 - 2,
                fill=OBSTACLE,
                outline=""
            )

        # Normal food
        if self.food is not None:

            x1, y1, x2, y2 = (
                self.cell_rect(self.food)
            )

            self.canvas.create_oval(
                x1 + 4,
                y1 + 4,
                x2 - 4,
                y2 - 4,
                fill=FOOD,
                outline=""
            )

        # Special food
        if self.special_food is not None:

            x1, y1, x2, y2 = (
                self.cell_rect(
                    self.special_food
                )
            )

            self.canvas.create_oval(
                x1 + 3,
                y1 + 3,
                x2 - 3,
                y2 - 3,
                fill=SPECIAL_FOOD,
                outline=""
            )

            self.canvas.create_text(
                (x1 + x2) // 2,
                (y1 + y2) // 2,
                text="5",
                fill="white",
                font=(
                    "Segoe UI",
                    9,
                    "bold"
                )
            )

        # Snake
        for index, segment in enumerate(
            self.snake
        ):

            x1, y1, x2, y2 = (
                self.cell_rect(segment)
            )

            color = (
                SNAKE_HEAD
                if index == 0
                else SNAKE_BODY
            )

            self.canvas.create_rectangle(
                x1 + 2,
                y1 + 2,
                x2 - 2,
                y2 - 2,
                fill=color,
                outline=""
            )

        # Pause overlay
        if self.paused:

            self.canvas.create_rectangle(
                0,
                0,
                WIDTH,
                HEIGHT,
                fill=BG,
                stipple="gray50",
                outline=""
            )

            self.canvas.create_text(
                WIDTH // 2,
                HEIGHT // 2,
                text=(
                    "PAUSED\n"
                    "Press SPACE to resume"
                ),
                fill=TEXT,
                font=(
                    "Segoe UI",
                    22,
                    "bold"
                ),
                justify="center"
            )

        # Game over overlay
        elif self.game_over:

            self.canvas.create_rectangle(
                0,
                0,
                WIDTH,
                HEIGHT,
                fill=BG,
                stipple="gray50",
                outline=""
            )

            self.canvas.create_text(
                WIDTH // 2,
                HEIGHT // 2,
                text=(
                    "GAME OVER\n"
                    "Press R to restart"
                ),
                fill=TEXT,
                font=(
                    "Segoe UI",
                    22,
                    "bold"
                ),
                justify="center"
            )

    # ---------------- CLOSE ----------------

    def close(self):

        if self.after_id is not None:

            try:

                self.root.after_cancel(
                    self.after_id
                )

            except tk.TclError:
                pass

        self.root.destroy()


# -------------------- START GAME --------------------

if __name__ == "__main__":

    root = tk.Tk()

    game = SnakeGame(root)

    root.mainloop()