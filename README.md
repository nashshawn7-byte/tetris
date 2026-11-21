import curses
import random
import time
from pathlib import Path

WIDTH = 10
HEIGHT = 20
HIDDEN = 2
TOTAL_HEIGHT = HEIGHT + HIDDEN
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
    1: curses.COLOR_CYAN,
    2: curses.COLOR_YELLOW,
    3: curses.COLOR_MAGENTA,
    4: curses.COLOR_GREEN,
    5: curses.COLOR_RED,
    6: curses.COLOR_BLUE,
    7: curses.COLOR_WHITE,
}

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


def put(stdscr, y, x, text, attr=0):
    height, width = stdscr.getmaxyx()
    if y < 0 or y >= height or x < 0 or x >= width:
        return
    try:
        stdscr.addstr(y, x, text[: width - x - 1], attr)
    except curses.error:
        pass


def draw_cell(stdscr, y, x, kind, ghost=False):
    attr = curses.A_DIM if ghost else curses.color_pair(kind)
    put(stdscr, y, x, "[]" if ghost else "  ", attr)


def draw_piece(stdscr, y0, x0, kind, rotation=0, ghost=False):
    matrix = ROTATIONS[kind][rotation]
    for y, row in enumerate(matrix):
        for x, char in enumerate(row):
            if char != ".":
                draw_cell(stdscr, y0 + y, x0 + x * 2, kind, ghost)


def draw(stdscr, game, paused):
    stdscr.erase()
    height, width = stdscr.getmaxyx()
    if height < 25 or width < 58:
        put(stdscr, height // 2, 1, "Terminal too small: need at least 58x25", curses.A_BOLD)
        return

    bx, by = 2, 2
    put(stdscr, by, bx + 6, "TETRIS", curses.A_BOLD)
    for row in range(HEIGHT + 2):
        put(stdscr, by + 1 + row, bx, "|" + " " * (WIDTH * 2 + 2) + "|")
    put(stdscr, by + 1, bx + 1, "_" * (WIDTH * 2 + 2))
    put(stdscr, by + HEIGHT + 2, bx + 1, "-" * (WIDTH * 2 + 2))

    for y in range(HIDDEN, TOTAL_HEIGHT):
        for x in range(WIDTH):
            kind = game.board.grid[y][x]
            if kind:
                draw_cell(stdscr, by + 2 + y - HIDDEN, bx + 2 + x * 2, kind)

    if game.current and not game.game_over:
        piece = game.current
        ghost_y = game.board.ghost_y(piece)
        original_y = piece.y
        piece.y = ghost_y
        if ghost_y != original_y:
            draw_piece(stdscr, by + 2, bx + 2, piece.kind, piece.rotation, ghost=True)
        piece.y = original_y
        draw_piece(stdscr, by + 2, bx + 2, piece.kind, piece.rotation)

    sx = bx + WIDTH * 2 + 7
    put(stdscr, by + 1, sx, "SCORE", curses.A_BOLD)
    put(stdscr, by + 2, sx, str(game.score).rjust(10))
    put(stdscr, by + 4, sx, "HIGH", curses.A_BOLD)
    put(stdscr, by + 5, sx, str(game.high_score).rjust(10))
    put(stdscr, by + 7, sx, "LINES", curses.A_BOLD)
    put(stdscr, by + 8, sx, str(game.lines).rjust(10))
    put(stdscr, by + 10, sx, "LEVEL", curses.A_BOLD)
    put(stdscr, by + 11, sx, str(game.level).rjust(10))

    put(stdscr, by + 13, sx, "HOLD", curses.A_BOLD)
    if game.held:
        draw_piece(stdscr, by + 14, sx, game.held)

    put(stdscr, by + 18, sx, "NEXT", curses.A_BOLD)
    for i, kind in enumerate(game.queue[:3]):
        draw_piece(stdscr, by + 19 + i * 4, sx, kind)

    controls = [
        "Left/Right or A/D  Move",
        "Up or X           Rotate CW",
        "Z                 Rotate CCW",
        "Down or S         Soft drop",
        "Space             Hard drop",
        "C                 Hold",
        "P                 Pause",
        "R                 Restart",
        "Q                 Quit",
    ]
    for i, line in enumerate(controls):
        put(stdscr, by + 2 + i, sx + 16, line)

    if paused:
        put(stdscr, by + HEIGHT // 2, bx + 6, "PAUSED", curses.A_BOLD | curses.A_REVERSE)
    if game.game_over:
        put(stdscr, by + HEIGHT // 2 - 1, bx + 3, "GAME OVER", curses.A_BOLD | curses.A_REVERSE)
        put(stdscr, by + HEIGHT // 2 + 1, bx + 1, "Press R to play again", curses.A_BOLD)


def key_action(key, game, state):
    if key in (ord("q"), ord("Q")):
        state["quit"] = True
        return
    if key in (ord("r"), ord("R")):
        state["game"] = Game(max(game.high_score, game.score))
        return
    if key in (ord("p"), ord("P"), 27):
        state["paused"] = not state["paused"]
        return
    if state["paused"] or game.game_over:
        return
    if key == curses.KEY_LEFT or key in (ord("a"), ord("A")):
        game.move(-1)
    elif key == curses.KEY_RIGHT or key in (ord("d"), ord("D")):
        game.move(1)
    elif key in (curses.KEY_UP, ord("x"), ord("X")):
        game.rotate(1)
    elif key in (ord("z"), ord("Z")):
        game.rotate(-1)
    elif key == curses.KEY_DOWN or key in (ord("s"), ord("S")):
        game.soft_drop()
    elif key == ord(" "):
        game.hard_drop()
    elif key in (ord("c"), ord("C")):
        game.hold()


def main(stdscr):
    curses.curs_set(0)
    stdscr.nodelay(True)
    if curses.has_colors():
        curses.start_color()
        for kind, color in COLORS.items():
            curses.init_pair(kind, color, curses.COLOR_BLACK)

    high_score = load_high_score()
    state = {"game": Game(high_score), "paused": False, "quit": False}
    last = time.monotonic()
    while not state["quit"]:
        now = time.monotonic()
        dt = min(100, (now - last) * 1000)
        last = now
        key = stdscr.getch()
        if key != curses.ERR:
            key_action(key, state["game"], state)
        game = state["game"]
        if not state["paused"]:
            game.update(dt)
        draw(stdscr, game, state["paused"])
        stdscr.refresh()
        time.sleep(0.016)

    high_score = max(high_score, state["game"].score)
    save_high_score(high_score)


if __name__ == "__main__":
    try:
        curses.wrapper(main)
    except KeyboardInterrupt:
        passgit remote add origin https://github.com/nashshawn7-byte/tetris.git
git branch -M main
git push -u origin main# tetris
another python tetris code
