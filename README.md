import random
import time
import tkinter as tk
from pathlib import Path

WIDTH = 12
HEIGHT = 20
HIDDEN = 2
TOTAL_HEIGHT = HEIGHT + HIDDEN
CELL_SIZE = 20
LOCK_DELAY = 500
MAX_LOCK_RESETS = 15

SHAPES = {
    1: ["....", "IIII", "....", "...."],
    2: ["OO", "OO"],
    3: [".T.", "TTT", "..."],
    4: [".SS", "SS.", "..."],
    5: ["ZZ.", ".ZZ", "..."],
    6: ["J..", "JJJ", "..."],
    7: ["..L", "LLL", "..."],
}

COLORS = {
    1: "#00d7d7",
    2: "#f4d03f",
    3: "#c850c0",
    4: "#52c878",
    5: "#e74c3c",
    6: "#4a90d9",
    7: "#bfc9ca",
}
LOCKED_COLOR = "#000080"

LINE_SCORES = {1: 100, 2: 300, 3: 500, 4: 800}
HIGH_SCORE_FILE = Path.home() / ".tetris_highscore"


def rotate_cw(matrix):
    n = len(matrix)
    return ["".join(matrix[n - 1 - c][r] for c in range(n)) for r in range(n)]


def rotate_ccw(matrix):
    n = len(matrix)
    return ["".join(matrix[c][n - 1 - r] for c in range(n)) for r in range(n)]


ROTATIONS = {}
for _kind, _shape in SHAPES.items():
    _states = [_shape]
    for _ in range(3):
        _states.append(rotate_cw(_states[-1]))
    ROTATIONS[_kind] = _states

JLSTZ_KICKS = {
    (0, 1): [(0, 0), (-1, 0), (-1, 1), (0, -2), (-1, -2)],
    (1, 0): [(0, 0), (1, 0), (1, -1), (0, 2), (1, 2)],
    (1, 2): [(0, 0), (1, 0), (1, -1), (0, 2), (1, 2)],
    (2, 1): [(0, 0), (-1, 0), (-1, 1), (0, -2), (-1, -2)],
    (2, 3): [(0, 0), (1, 0), (1, 1), (0, -2), (1, -2)],
    (3, 2): [(0, 0), (-1, 0), (-1, -1), (0, 2), (-1, 2)],
    (3, 0): [(0, 0), (-1, 0), (-1, -1), (0, 2), (-1, 2)],
    (0, 3): [(0, 0), (1, 0), (1, 1), (0, -2), (1, -2)],
}

I_KICKS = {
    (0, 1): [(0, 0), (-2, 0), (1, 0), (-2, -1), (1, 2)],
    (1, 0): [(0, 0), (2, 0), (-1, 0), (2, 1), (-1, -2)],
    (1, 2): [(0, 0), (-1, 0), (2, 0), (-1, 2), (2, -1)],
    (2, 1): [(0, 0), (1, 0), (-2, 0), (1, -2), (-2, 1)],
    (2, 3): [(0, 0), (2, 0), (-1, 0), (2, 1), (-1, -2)],
    (3, 2): [(0, 0), (-2, 0), (1, 0), (-2, -1), (1, 2)],
    (3, 0): [(0, 0), (1, 0), (-2, 0), (1, -2), (-2, 1)],
    (0, 3): [(0, 0), (-1, 0), (2, 0), (-1, 2), (2, -1)],
}


def gravity_interval(level):
    base = max(0.1, 0.8 - (level - 1) * 0.007)
    return max(50, int(1000 * base ** (level - 1)))


def load_high_score():
    try:
        return max(0, int(HIGH_SCORE_FILE.read_text().strip()))
    except (OSError, ValueError):
        return 0


def save_high_score(score):
    try:
        HIGH_SCORE_FILE.write_text(str(score))
    except OSError:
        pass


class Piece:
    def __init__(self, kind):
        self.kind = kind
        self.rotation = 0
        self.x = (WIDTH - len(ROTATIONS[kind][0])) // 2
        self.y = HIDDEN if kind == 2 else HIDDEN - 1

    def cells(self, rotation=None):
        matrix = ROTATIONS[self.kind][self.rotation if rotation is None else rotation]
        for y, row in enumerate(matrix):
            for x, char in enumerate(row):
                if char != ".":
                    yield self.x + x, self.y + y


class Board:
    def __init__(self):
        self.grid = [[0] * WIDTH for _ in range(TOTAL_HEIGHT)]

    def collides(self, piece, dx=0, dy=0, rotation=None):
        for x, y in piece.cells(rotation):
            px, py = x + dx, y + dy
            if px < 0 or px >= WIDTH or py < 0 or py >= TOTAL_HEIGHT:
                return True
            if self.grid[py][px]:
                return True
        return False

    def lock(self, piece):
        for x, y in piece.cells():
            self.grid[y][x] = piece.kind
        cleared = 0
        row = TOTAL_HEIGHT - 1
        while row >= 0:
            if all(self.grid[row]):
                del self.grid[row]
                self.grid.insert(0, [0] * WIDTH)
                cleared += 1
            else:
                row -= 1
        return cleared

    def ghost_y(self, piece):
        dy = 0
        while not self.collides(piece, dy=dy + 1):
            dy += 1
        return piece.y + dy


class Game:
    def __init__(self, high_score=0):
        self.board = Board()
        self.bag = []
        self.queue = [self._from_bag() for _ in range(5)]
        self.current = None
        self.held = None
        self.can_hold = True
        self.score = 0
        self.lines = 0
        self.level = 1
        self.high_score = high_score
        self.combo = -1
        self.back_to_back = False
        self.fall_time = 0
        self.lock_time = 0
        self.lock_resets = 0
        self.game_over = False
        self.spawn()

    def _from_bag(self):
        if not self.bag:
            self.bag = list(SHAPES)
            random.shuffle(self.bag)
        return self.bag.pop()

    def spawn(self, kind=None):
        if kind is None:
            kind = self.queue.pop(0)
            self.queue.append(self._from_bag())
        self.current = Piece(kind)
        self.fall_time = 0
        self.lock_time = 0
        self.lock_resets = 0
        if self.board.collides(self.current):
            self.game_over = True

    def _reset_lock(self):
        if self.lock_time and self.lock_resets < MAX_LOCK_RESETS:
            self.lock_time = 0
            self.lock_resets += 1

    def move(self, dx):
        if self.game_over or self.board.collides(self.current, dx=dx):
            return False
        self.current.x += dx
        self._reset_lock()
        return True

    def rotate(self, direction):
        if self.game_over:
            return False
        piece = self.current
        if piece.kind == 2:
            return False
        old, new = piece.rotation, (piece.rotation + direction) % 4
        kicks = I_KICKS if piece.kind == 1 else JLSTZ_KICKS
        for dx, dy in kicks[(old, new)]:
            if not self.board.collides(piece, dx=dx, dy=-dy, rotation=new):
                piece.x += dx
                piece.y -= dy
                piece.rotation = new
                self._reset_lock()
                return True
        return False

    def move_down(self):
        if self.board.collides(self.current, dy=1):
            return False
        self.current.y += 1
        self.lock_time = 0
        return True

    def soft_drop(self):
        if self.game_over:
            return
        if self.move_down():
            self.score += 1
        self.fall_time = 0

    def hard_drop(self):
        if self.game_over:
            return
        distance = self.board.ghost_y(self.current) - self.current.y
        self.current.y += distance
        self.score += distance * 2
        self.lock()

    def hold(self):
        if self.game_over or not self.can_hold:
            return
        kind = self.current.kind
        if self.held is None:
            self.held = kind
            self.spawn()
        else:
            self.held, kind = kind, self.held
            self.spawn(kind)
        self.can_hold = False

    def lock(self):
        piece = self.current
        if all(y < HIDDEN for _, y in piece.cells()):
            self.game_over = True
            return
        cleared = self.board.lock(piece)
        if cleared:
            self.combo += 1
            self.back_to_back = cleared == 4 and self.back_to_back
            points = LINE_SCORES[cleared] * self.level
            if self.back_to_back:
                points = int(points * 1.5)
            if self.combo > 0:
                points += 50 * self.combo * self.level
            self.score += points
            self.lines += cleared
            self.level = self.lines // 10 + 1
            self.back_to_back = cleared == 4
        else:
            self.combo = -1
            self.back_to_back = False
        self.high_score = max(self.high_score, self.score)
        self.spawn()
        self.can_hold = True

    def update(self, dt):
        if self.game_over:
            return
        self.fall_time += dt
        interval = gravity_interval(self.level)
        while self.fall_time >= interval:
            self.fall_time -= interval
            if not self.move_down():
                self.fall_time = 0
                break
        if self.board.collides(self.current, dy=1):
            self.lock_time += dt
            if self.lock_time >= LOCK_DELAY:
                self.lock()
        else:
            self.lock_time = 0


def draw_piece(canvas, game_x, game_y, kind, rotation=0, ghost=False, cell_size=CELL_SIZE):
    matrix = ROTATIONS[kind][rotation]
    for row_index, row in enumerate(matrix):
        for column_index, char in enumerate(row):
            if char == ".":
                continue
            board_x = game_x + column_index
            board_y = game_y + row_index
            if board_y < HIDDEN:
                continue
            canvas_x = 20 + board_x * cell_size
            canvas_y = 20 + (board_y - HIDDEN) * cell_size
            fill = "#333333" if ghost else COLORS[kind]
            outline = COLORS[kind] if ghost else "#1b1b1b"
            canvas.create_rectangle(
                canvas_x,
                canvas_y,
                canvas_x + cell_size - 2,
                canvas_y + cell_size - 2,
                fill=fill,
                outline=outline,
                width=2,
            )


def draw_preview_piece(canvas, kind, offset_x, offset_y):
    matrix = ROTATIONS[kind][0]
    for row_index, row in enumerate(matrix):
        for column_index, char in enumerate(row):
            if char != ".":
                canvas.create_rectangle(
                    offset_x + column_index * 20,
                    offset_y + row_index * 20,
                    offset_x + column_index * 20 + 18,
                    offset_y + row_index * 20 + 18,
                    fill=LOCKED_COLOR,
                    outline="#1b1b1b",
                )


def main():
    root = tk.Tk()
    root.title("Tetris")
    root.resizable(False, False)

    canvas = tk.Canvas(
        root,
        width=520,
        height=520,
        bg="#111111",
        highlightthickness=0,
    )
    canvas.pack(side=tk.LEFT)

    side = tk.Canvas(
        root,
        width=180,
        height=520,
        bg="#222222",
        highlightthickness=0,
    )
    side.pack(side=tk.LEFT)

    high_score = load_high_score()
    state = {"game": Game(high_score), "paused": False, "quit": False}

    def restart():
        game = state["game"]
        state["game"] = Game(max(game.high_score, game.score))

    def key_action(event):
        game = state["game"]
        key = event.keysym.lower()
        char = event.char.lower()

        if key == "q" or char == "q":
            state["quit"] = True
            root.destroy()
            return
        if key == "r" or char == "r":
            restart()
            return
        if key == "p" or key == "escape" or char == "p":
            state["paused"] = not state["paused"]
            return
        if state["paused"] or game.game_over:
            return

        if key == "left" or char == "a":
            game.move(-1)
        elif key == "right" or char == "d":
            game.move(1)
        elif key == "up" or char == "x":
            game.rotate(1)
        elif char == "z":
            game.rotate(-1)
        elif key == "down" or char == "s":
            game.soft_drop()
        elif key == "space":
            game.hard_drop()
        elif char == "c":
            game.hold()

    root.bind("<Key>", key_action)

    def draw():
        canvas.delete("all")
        game = state["game"]

        canvas.create_rectangle(
            20,
            20,
            20 + WIDTH * CELL_SIZE,
            20 + HEIGHT * CELL_SIZE,
            fill="#181818",
            outline="#aaaaaa",
            width=2,
        )

        for row in range(HEIGHT):
            for column in range(WIDTH):
                kind = game.board.grid[row + HIDDEN][column]
                if kind:
                    canvas.create_rectangle(
                        20 + column * CELL_SIZE,
                        20 + row * CELL_SIZE,
                        20 + column * CELL_SIZE + CELL_SIZE - 2,
                        20 + row * CELL_SIZE + CELL_SIZE - 2,
                        fill=LOCKED_COLOR,
                        outline="#1b1b1b",
                    )

        if game.current and not game.game_over:
            piece = game.current
            ghost_y = game.board.ghost_y(piece)
            if ghost_y != piece.y:
                draw_piece(
                    canvas,
                    piece.x,
                    ghost_y,
                    piece.kind,
                    piece.rotation,
                    ghost=True,
                )
            draw_piece(
                canvas,
                piece.x,
                piece.y,
                piece.kind,
                piece.rotation,
            )

        side.delete("all")
        side.create_text(90, 20, text="TETRIS", fill="#ffffff", font=("Arial", 18, "bold"))
        side.create_text(90, 55, text=f"SCORE\n{game.score}", fill="#ffffff", font=("Arial", 12), justify="center")
        side.create_text(90, 105, text=f"HIGH\n{game.high_score}", fill="#ffffff", font=("Arial", 12), justify="center")
        side.create_text(90, 155, text=f"LINES\n{game.lines}", fill="#ffffff", font=("Arial", 12), justify="center")
        side.create_text(90, 205, text=f"LEVEL\n{game.level}", fill="#ffffff", font=("Arial", 12), justify="center")
        side.create_text(90, 245, text="HOLD", fill="#ffffff", font=("Arial", 12, "bold"))
        if game.held:
            draw_preview_piece(side, game.held, 70, 265)
        side.create_text(90, 355, text="NEXT", fill="#ffffff", font=("Arial", 12, "bold"))
        for index, kind in enumerate(game.queue[:3]):
            draw_preview_piece(side, kind, 70, 375 + index * 42)

        controls = [
            "Arrows/A/D move",
            "Up/X rotate CW",
            "Z rotate CCW",
            "S soft drop",
            "Space hard drop",
            "C hold",
            "P pause",
            "R restart",
            "Q quit",
        ]
        for index, line in enumerate(controls):
            side.create_text(90, 295 + index * 18, text=line, fill="#cccccc", font=("Arial", 9))

        if state["paused"]:
            canvas.create_text(
                140,
                250,
                text="PAUSED",
                fill="#ffffff",
                font=("Arial", 24, "bold"),
            )
        if game.game_over:
            canvas.create_text(
                140,
                230,
                text="GAME OVER",
                fill="#ffffff",
                font=("Arial", 24, "bold"),
            )
            canvas.create_text(
                140,
                260,
                text="Press R to restart",
                fill="#ffffff",
                font=("Arial", 12),
            )

    last = time.monotonic()

    def update():
        nonlocal last
        now = time.monotonic()
        dt = min(100, (now - last) * 1000)
        last = now
        game = state["game"]
        if not state["paused"]:
            game.update(dt)
        draw()
        if not state["quit"]:
            root.after(16, update)

    def on_close():
        state["quit"] = True
        save_high_score(max(state["game"].high_score, state["game"].score))
        root.destroy()

    root.protocol("WM_DELETE_WINDOW", on_close)
    update()
    root.mainloop()


if __name__ == "__main__":
    main()
