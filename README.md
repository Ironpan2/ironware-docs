# ironware — Plugin Developer Guide

Everything you need to add new features by dropping a `.py` file into `features/`.

---

## Quick Start

1. Create `features/myplugin.py`
2. Subclass `Plugin` from `core.plugin`
3. Add `plugin = MyPlugin()` at the bottom
4. Launch ironware — it loads automatically, no other files touched

```python
from core.plugin import Plugin

class MyPlugin(Plugin):
    PLUGIN_NAME = "My Plugin"
    PLUGIN_TAB  = "Misc"

    def register(self, builder, mem, settings):
        builder.add_card("My Feature")
        builder.add_toggle("Enable", "myplugin_enabled", default=False)

    def draw(self, mem, settings):
        if not settings.get("myplugin_enabled"):
            return
        # runs every overlay frame

plugin = MyPlugin()
```

---

## Plugin Class Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `PLUGIN_NAME` | `str` | Display name shown in console on load |
| `PLUGIN_TAB` | `str \| None` | Tab to add UI to: `"Visuals"` `"Aimbot"` `"Misc"` or `None` |
| `PLUGIN_TOOL_BUTTON` | `str \| None` | e.g. `"🔧  My Tool"` — adds a sidebar button that opens a window |

---

## Plugin Methods

### `register(builder, mem, settings)`
Called **once at startup**. Use `builder` to add UI widgets.

### `draw(mem, settings)`
Called **every overlay frame** inside `begin_drawing()` / `end_drawing()`.
Read settings, write memory, call `pm.draw_*` here.

### `on_stop(mem)`
Called when overlay stops. Reset memory writes and stop threads.

### `open_tool(mem)`
Called when the sidebar tool button is clicked. Only needed if `PLUGIN_TOOL_BUTTON` is set.

---

## UIBuilder Reference

### `add_card(title="")`
Opens a dark bordered section. All widgets go inside until the next `add_card()`.
```python
builder.add_card("My Section")
builder.add_card()   # card with no title
```

### `add_toggle(label, key, default=False, on_change=None)`
ON/OFF switch. Stores `bool` in `settings[key]`.
```python
# Simple toggle — read with settings.get("my_key")
builder.add_toggle("Show Boxes", "esp_boxes", default=True)

# With callback — fires immediately when user flips the switch
builder.add_toggle("Enable WalkSpeed", "ws_enabled", default=False,
                   on_change=lambda enabled: reset_ws() if not enabled else None)

# Returns a handle so you can flip it from code (e.g. a reset button)
handle = builder.add_toggle("Enable", "my_key", default=False)
handle.set(False)   # turns it off programmatically
```

### `add_slider(label, key, from_, to, fmt, default)`
Horizontal slider. Stores `float` in `settings[key]`.
```python
builder.add_slider("Speed",     "speed",  from_=1,    to=100,  fmt="{:.0f}",   default=10)
builder.add_slider("Smoothing", "smooth", from_=0.05, to=1.0,  fmt="{:.2f}",   default=0.35)
builder.add_slider("FOV",       "fov",    from_=50,   to=1000, fmt="{:.0f}px", default=300)
```

### `add_entry(label, key, default=0, on_set=None)`
Text box + SET button. Stores `float` in `settings[key]`.
```python
# Just store value, read it in draw()
builder.add_entry("WalkSpeed  (default 16)", "ws_value", default=50)

# Callback fires immediately when SET is clicked
builder.add_entry("WalkSpeed", "ws_value", default=50,
                  on_set=lambda v: apply_walkspeed(mem, v))
```

### `add_dropdown(label, key, options, default=None)`
Dropdown selector. Stores selected `str` in `settings[key]`.
```python
builder.add_dropdown("Key", "aim_key",
                     options=["Mouse4", "Mouse5", "RMB", "CapsLock"],
                     default="Mouse4")

# Read in draw():
key = settings.get("aim_key", "Mouse4")
```

### `add_button(label, callback, color=None, hover=None)`
Full-width button. Callback runs on the GUI thread — don't block.
```python
builder.add_button("Reset",      self._reset)
builder.add_button("Delete All", self._nuke, color="#7f1d1d", hover="#991b1b")
```

### `add_divider()` / `add_label(text)` / `add_space()`
```python
builder.add_divider()                    # thin horizontal line
builder.add_label("Some hint text")      # small muted grey text
builder.add_space()                      # small vertical gap
```

---

## Memory API

### Reading
```python
mem.read_float(address)           # float | None
mem.read_int(address)             # int | None    (4 bytes signed)
mem.read_int64(address)           # int | None    (8 bytes signed)
mem.read_ptr(address)             # int | None    (pointer)
mem.read_memory(address, size)    # bytes | None  (raw)
mem.read_string(address)          # str           (Roblox std::string)
```

### Writing
```python
mem.write_float(address, value)   # write a 4-byte float
mem.write(address, bytes_data)    # write raw bytes
```

### Instance tree
```python
mem.get_children(parent_ptr)           # list[int]
mem.get_name(ptr)                      # str
mem.get_class(ptr)                     # str
mem.find_child(parent, "PartName")     # int | None  — find by name
mem.find_class(parent, "Humanoid")     # int | None  — find by class
```

### World / screen
```python
mem.read_pos(part_ptr)        # vec3 | None   world position of a Part
mem.world_to_screen(vec3)     # vec2          screen pixel coords
                              #               returns (-1,-1) if off screen
mem.get_viewport()            # vec2          window size in pixels
```

### Useful attributes
```python
mem.local_player    # your Player instance ptr
mem.players         # Players service ptr
mem.data_model      # DataModel ptr
mem.base_address    # RobloxPlayerBeta.exe base address
```

---

## Player Dict

`mem.get_players()` returns a list — one dict per enemy player:

```python
{
    "player_name":            str,
    "player_ptr":             int,        # Player instance
    "character_ptr":          int,        # character Model
    "humanoid_root_part_ptr": int,        # HumanoidRootPart
    "root_pos":               vec3,       # HRP world position
    "head_pos":               vec3,       # Head world position
    "player_size":            vec3,       # HRP part dimensions
    "health":                 float|None,
    "max_health":             float|None,
    "rig_type":               int,        # 0 = R6   1 = R15
    "body_parts":             dict,       # part name → vec3 world position
}
```

**R15 body_parts keys:** `Head UpperTorso LowerTorso LeftUpperArm LeftLowerArm LeftHand RightUpperArm RightLowerArm RightHand LeftUpperLeg LeftLowerLeg LeftFoot RightUpperLeg RightLowerLeg RightFoot HumanoidRootPart`

**R6 body_parts keys:** `Head Torso Left Arm Right Arm Left Leg Right Leg HumanoidRootPart`

---

## Drawing (pyMeow)

Only call these inside `draw()`.

```python
import pyMeow as pm

# Colors
pm.get_color("white")    pm.get_color("red")      pm.get_color("green")
pm.get_color("yellow")   pm.get_color("blue")     pm.get_color("orange")
pm.get_color("purple")   pm.get_color("darkgray")

# Shapes
pm.draw_rectangle(x, y, w, h, color)            # filled rectangle
pm.draw_rectangle_lines(x, y, w, h, color)      # outline rectangle
pm.draw_circle(x, y, radius, color)             # filled circle
pm.draw_circle_lines(x, y, radius, color)       # circle outline
pm.draw_line(x1, y1, x2, y2, color)             # line between two points

# Text
pm.draw_text("hello", x, y, font_size, color)
```

---

## Offsets

```python
from offsets import OFFSETS

ws_off   = int(OFFSETS["WalkSpeed"],     16)
jp_off   = int(OFFSETS["JumpPower"],     16)
hp_off   = int(OFFSETS["Health"],        16)
pos_off  = int(OFFSETS["Position"],      16)
prim_off = int(OFFSETS["Primitive"],     16)
vel_off  = int(OFFSETS["Velocity"],      16)
cf_off   = int(OFFSETS.get("CFrame", "0xC0"), 16)
mi_off   = int(OFFSETS["ModelInstance"], 16)
```

---

---

# Examples

---

## Example 1 — WalkSpeed Changer

Writes walkspeed every frame so it survives respawns automatically.
Shows toggle + entry + on_change callback + reset button.

```python
from core.plugin import Plugin
from offsets     import OFFSETS

DEFAULT_WS = 16.0   # Roblox default walkspeed


def _get_humanoid(mem):
    """Walk local_player → Character → Humanoid. Returns ptr or None."""
    lp = getattr(mem, "local_player", None)
    if not lp:
        return None
    try:
        char = mem.read_ptr(lp + int(OFFSETS["ModelInstance"], 16))
        if not char or mem.get_class(char) != "Model":
            return None
        return mem.find_class(char, "Humanoid")
    except Exception:
        return None


def _write_ws(mem, value: float):
    """Write to both WalkSpeed and WalkSpeedCheck (both needed)."""
    h = _get_humanoid(mem)
    if not h:
        return
    v = round(value)   # Roblox expects integer walkspeed
    try:
        mem.write_float(h + int(OFFSETS["WalkSpeed"],      16), v)
        mem.write_float(h + int(OFFSETS["WalkSpeedCheck"], 16), v)
    except Exception:
        pass


class WalkSpeedPlugin(Plugin):
    PLUGIN_NAME = "WalkSpeed"
    PLUGIN_TAB  = "Misc"

    def register(self, builder, mem, settings):
        # Store mem reference for use in button callbacks
        self._mem = mem

        builder.add_card("WalkSpeed")

        # Toggle — when turned OFF, immediately resets to default 16
        self._toggle = builder.add_toggle(
            "Enable",
            "ws_enabled",
            default=False,
            on_change=lambda v: _write_ws(mem, DEFAULT_WS) if not v else None
        )

        builder.add_divider()

        # Entry field — user types a number and clicks SET
        # on_set fires immediately when SET is clicked so it applies right away
        builder.add_entry(
            f"Speed   (default {int(DEFAULT_WS)})",
            "ws_value",
            default=100,
            on_set=lambda v: _write_ws(mem, v)
        )

        builder.add_divider()

        builder.add_button("Reset to Default", self._reset)
        builder.add_label(f"Default Roblox walkspeed is {int(DEFAULT_WS)}")
        builder.add_space()

    def draw(self, mem, settings):
        # Re-apply every frame — if player respawns they get a fresh Humanoid
        # and the value resets. Writing every frame means it restores within
        # one frame automatically without any extra logic.
        if settings.get("ws_enabled", False):
            _write_ws(mem, settings.get("ws_value", DEFAULT_WS))

    def on_stop(self, mem):
        # Always clean up when overlay closes
        _write_ws(mem, DEFAULT_WS)

    def _reset(self):
        """Called by the Reset button."""
        _write_ws(self._mem, DEFAULT_WS)
        if self._toggle:
            self._toggle.set(False)  # flip the switch back to OFF


plugin = WalkSpeedPlugin()
```

---

## Example 2 — Jump Power Changer (Slider version)

Same pattern as walkspeed but uses a slider for smoother control.

```python
from core.plugin import Plugin
from offsets     import OFFSETS

DEFAULT_JP = 50.0


def _get_humanoid(mem):
    lp = getattr(mem, "local_player", None)
    if not lp:
        return None
    try:
        char = mem.read_ptr(lp + int(OFFSETS["ModelInstance"], 16))
        if not char or mem.get_class(char) != "Model":
            return None
        return mem.find_class(char, "Humanoid")
    except Exception:
        return None


class JumpPowerPlugin(Plugin):
    PLUGIN_NAME = "Jump Power"
    PLUGIN_TAB  = "Misc"

    def register(self, builder, mem, settings):
        self._mem = mem

        builder.add_card("Jump Power")

        self._toggle = builder.add_toggle(
            "Enable",
            "jp_enabled",
            default=False,
            on_change=lambda v: self._reset_jp() if not v else None
        )

        builder.add_divider()

        # Slider from 50 (default) up to 1000
        # Value is read in draw() each frame
        builder.add_slider(
            "Power",
            "jp_value",
            from_=50,
            to=1000,
            fmt="{:.0f}",
            default=200
        )

        builder.add_divider()
        builder.add_button("Reset to Default", self._reset_jp)
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("jp_enabled", False):
            return
        h = _get_humanoid(mem)
        if not h:
            return
        try:
            mem.write_float(h + int(OFFSETS["JumpPower"], 16),
                            settings.get("jp_value", DEFAULT_JP))
        except Exception:
            pass

    def on_stop(self, mem):
        self._reset_jp()

    def _reset_jp(self):
        h = _get_humanoid(self._mem)
        if h:
            try:
                self._mem.write_float(h + int(OFFSETS["JumpPower"], 16), DEFAULT_JP)
            except Exception:
                pass
        if self._toggle:
            self._toggle.set(False)


plugin = JumpPowerPlugin()
```

---

## Example 3 — Draw Shapes on Screen

Every pyMeow draw call demonstrated. Shows rectangles, circles, lines, and
text positioned relative to the screen center and relative to player positions.

```python
import pyMeow as pm
from core.plugin import Plugin
from mem         import vec3


class ShapeDrawerPlugin(Plugin):
    PLUGIN_NAME = "Shape Drawer"
    PLUGIN_TAB  = "Visuals"

    def register(self, builder, mem, settings):
        builder.add_card("Shape Examples")
        builder.add_toggle("Show All Shapes",  "shapes_all",      default=False)
        builder.add_divider()
        builder.add_toggle("Crosshair",        "shape_crosshair", default=True)
        builder.add_divider()
        builder.add_toggle("Screen Border",    "shape_border",    default=True)
        builder.add_divider()
        builder.add_toggle("Dot on Players",   "shape_pdots",     default=True)
        builder.add_divider()
        builder.add_toggle("Text Label Demo",  "shape_text",      default=True)
        builder.add_divider()
        builder.add_slider("Dot Size", "shape_dot_size",
                           from_=2, to=20, fmt="{:.0f}", default=6)
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("shapes_all", False):
            return

        vp       = mem.get_viewport()
        cx       = int(vp.x / 2)   # exact screen center X
        cy       = int(vp.y / 2)   # exact screen center Y
        dot_size = int(settings.get("shape_dot_size", 6))

        # ── Crosshair ─────────────────────────────────────────────────────────
        # draw_circle(x, y, radius, color) — filled circle
        # draw_circle_lines(x, y, radius, color) — hollow circle outline
        if settings.get("shape_crosshair", True):

            # Small filled dot at center
            pm.draw_circle(cx, cy, dot_size, pm.get_color("white"))

            # Hollow ring around it
            pm.draw_circle_lines(cx, cy, dot_size + 4, pm.get_color("white"))

            # Four lines extending outward (classic crosshair look)
            gap = dot_size + 6   # gap between dot edge and line start
            length = 12          # how long each line is

            pm.draw_line(cx - gap - length, cy, cx - gap, cy, pm.get_color("white"))  # left
            pm.draw_line(cx + gap, cy, cx + gap + length, cy, pm.get_color("white"))  # right
            pm.draw_line(cx, cy - gap - length, cx, cy - gap, pm.get_color("white"))  # up
            pm.draw_line(cx, cy + gap, cx, cy + gap + length, pm.get_color("white"))  # down

        # ── Screen border ──────────────────────────────────────────────────────
        # draw_rectangle_lines(x, y, width, height, color) — hollow rectangle
        # draw_rectangle(x, y, width, height, color) — filled rectangle
        if settings.get("shape_border", True):
            pm.draw_rectangle_lines(
                2, 2,                         # top-left corner
                int(vp.x) - 4,               # width
                int(vp.y) - 4,               # height
                pm.get_color("white")
            )

        # ── Dot on every visible player's head ───────────────────────────────
        if settings.get("shape_pdots", True):
            for p in mem.get_players():
                # Convert 3D world position to 2D screen pixel position
                screen = mem.world_to_screen(p["head_pos"])

                # world_to_screen returns (-1, -1) when off screen — always check
                if screen.x == -1:
                    continue

                sx = int(screen.x)
                sy = int(screen.y)

                # Filled red dot
                pm.draw_circle(sx, sy, dot_size, pm.get_color("red"))

                # White outline ring
                pm.draw_circle_lines(sx, sy, dot_size + 3, pm.get_color("white"))

                # Line from screen center to this player (a snapline)
                pm.draw_line(cx, int(vp.y), sx, sy, pm.get_color("white"))

        # ── Text drawing ───────────────────────────────────────────────────────
        # draw_text("string", x, y, font_size, color)
        # x/y is the TOP-LEFT of the text
        # Rough width estimate: each character ≈ font_size * 0.6 pixels wide
        if settings.get("shape_text", True):

            # Watermark in top-left corner
            # Dark background box first for readability
            label   = "ironware"
            font_sz = 12
            box_w   = len(label) * 7 + 10
            box_h   = 18
            pm.draw_rectangle(8, 8, box_w, box_h, pm.get_color("darkgray"))
            pm.draw_text(label, 13, 10, font_sz, pm.get_color("white"))

            # Centered text at the top of the screen
            msg    = "Shape Drawer Active"
            msg_x  = cx - len(msg) * 3   # roughly center it
            pm.draw_text(msg, msg_x, 14, 11, pm.get_color("yellow"))

            # Text above each player showing their name + health
            for p in mem.get_players():
                screen = mem.world_to_screen(p["head_pos"])
                if screen.x == -1:
                    continue

                # Build the string
                hp_str = ""
                if p["health"] is not None and p["max_health"]:
                    pct    = p["health"] / p["max_health"] * 100
                    hp_str = f"  {pct:.0f}%"

                text  = p["player_name"] + hp_str
                text_x = int(screen.x) - len(text) * 3  # center on player
                text_y = int(screen.y) - 20              # above the head dot

                pm.draw_text(text, text_x, text_y, 11, pm.get_color("white"))


plugin = ShapeDrawerPlugin()
```

---

## Example 4 — Player Info Above Heads

Draws name, health bar, and distance above every visible enemy.
Shows stacking multiple text lines and coloring health dynamically.

```python
import pyMeow as pm
import math
from core.plugin import Plugin
from mem         import vec3


class PlayerInfoPlugin(Plugin):
    PLUGIN_NAME = "Player Info"
    PLUGIN_TAB  = "Visuals"

    def register(self, builder, mem, settings):
        builder.add_card("Player Info")
        builder.add_toggle("Enable",       "pinfo_enabled",  default=False)
        builder.add_divider()
        builder.add_toggle("Name",         "pinfo_name",     default=True)
        builder.add_divider()
        builder.add_toggle("Health %",     "pinfo_health",   default=True)
        builder.add_divider()
        builder.add_toggle("Distance",     "pinfo_dist",     default=True)
        builder.add_divider()
        builder.add_toggle("Tool / Weapon","pinfo_tool",     default=True)
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("pinfo_enabled", False):
            return

        # Get our own position so we can calculate distance to each player
        local_pos = _get_local_pos(mem)

        for p in mem.get_players():
            screen = mem.world_to_screen(p["head_pos"])
            if screen.x == -1:
                continue

            # Build a list of (text, color) pairs
            lines = []

            if settings.get("pinfo_name", True):
                lines.append((p["player_name"], pm.get_color("white")))

            if settings.get("pinfo_health", True):
                hp  = p["health"]
                mhp = p["max_health"]
                if hp is not None and mhp and mhp > 0:
                    pct = hp / mhp * 100
                    if pct > 60:
                        hcol = pm.get_color("green")
                    elif pct > 30:
                        hcol = pm.get_color("yellow")
                    else:
                        hcol = pm.get_color("red")
                    lines.append((f"{int(hp)}/{int(mhp)}  ({pct:.0f}%)", hcol))

            if settings.get("pinfo_dist", True) and local_pos:
                r  = p["root_pos"]
                dx = r.x - local_pos.x
                dy = r.y - local_pos.y
                dz = r.z - local_pos.z
                dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                lines.append((f"{dist:.0f} studs", pm.get_color("white")))

            if settings.get("pinfo_tool", True):
                tool = _get_held_tool(mem, p["character_ptr"])
                if tool:
                    lines.append((f"[ {tool} ]", pm.get_color("yellow")))

            # Draw each line stacked upward from the head position
            line_h  = 14
            start_y = int(screen.y) - len(lines) * line_h - 6

            for i, (text, color) in enumerate(lines):
                tx = int(screen.x) - len(text) * 3   # roughly centered
                ty = start_y + i * line_h
                pm.draw_text(text, tx, ty, 11, color)


def _get_local_pos(mem):
    """Return our own world position or None."""
    from offsets import OFFSETS
    lp = getattr(mem, "local_player", None)
    if not lp:
        return None
    try:
        char = mem.read_ptr(lp + int(OFFSETS["ModelInstance"], 16))
        if not char:
            return None
        hrp = mem.find_child(char, "HumanoidRootPart")
        return mem.read_pos(hrp) if hrp else None
    except Exception:
        return None


def _get_held_tool(mem, character_ptr) -> str | None:
    """Return the name of the tool a player is holding, or None."""
    if not character_ptr:
        return None
    try:
        for child in mem.get_children(character_ptr):
            if mem.get_class(child) == "Tool":
                name = mem.get_name(child)
                if name:
                    return name
    except Exception:
        pass
    return None


plugin = PlayerInfoPlugin()
```

---

## Example 5 — FOV Circle + Closest Player Arrow

Draws a FOV circle and an arrow pointing at the closest enemy inside it.
Shows distance math, normalization, and multi-line drawing.

```python
import pyMeow as pm
import math
from core.plugin import Plugin


class FOVIndicatorPlugin(Plugin):
    PLUGIN_NAME = "FOV Indicator"
    PLUGIN_TAB  = "Visuals"

    def register(self, builder, mem, settings):
        builder.add_card("FOV Indicator")
        builder.add_toggle("Show FOV Circle",   "fov_ind_enabled",   default=False)
        builder.add_divider()
        builder.add_slider("Radius", "fov_ind_radius",
                           from_=50, to=800, fmt="{:.0f}px", default=300)
        builder.add_divider()
        builder.add_toggle("Arrow to Closest",  "fov_ind_arrow",     default=True)
        builder.add_label("Draws an arrow toward nearest enemy in FOV")
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("fov_ind_enabled", False):
            return

        vp     = mem.get_viewport()
        cx     = int(vp.x / 2)   # crosshair X
        cy     = int(vp.y / 2)   # crosshair Y
        radius = settings.get("fov_ind_radius", 300.0)

        # Draw the circle
        pm.draw_circle_lines(cx, cy, int(radius), pm.get_color("white"))

        if not settings.get("fov_ind_arrow", True):
            return

        # Find the closest player whose head is on screen
        best_dist   = float("inf")
        best_screen = None

        for p in mem.get_players():
            screen = mem.world_to_screen(p["head_pos"])
            if screen.x == -1:
                continue
            dx   = screen.x - cx
            dy   = screen.y - cy
            dist = math.sqrt(dx*dx + dy*dy)
            if dist < best_dist:
                best_dist   = dist
                best_screen = screen

        if best_screen is None:
            return

        # Only draw arrow if closest player is inside the FOV radius
        if best_dist > radius:
            return

        # Direction vector from crosshair to player
        dx = best_screen.x - cx
        dy = best_screen.y - cy
        length = math.sqrt(dx*dx + dy*dy)
        if length < 1:
            return

        # Normalize (make it length 1)
        nx = dx / length
        ny = dy / length

        # Arrow tip stops 10px before the player
        stop  = min(length - 10, radius - 5)
        tip_x = int(cx + nx * stop)
        tip_y = int(cy + ny * stop)

        # Arrow base is 25px behind the tip
        base_x = int(tip_x - nx * 25)
        base_y = int(tip_y - ny * 25)

        # Perpendicular vector for the arrowhead wings
        px = -ny * 7
        py =  nx * 7

        # Draw the two wings of the arrowhead
        pm.draw_line(tip_x, tip_y, int(base_x + px), int(base_y + py), pm.get_color("red"))
        pm.draw_line(tip_x, tip_y, int(base_x - px), int(base_y - py), pm.get_color("red"))

        # Draw the arrow shaft
        pm.draw_line(cx, cy, tip_x, tip_y, pm.get_color("red"))

        # Distance label next to the closest player
        label = f"{best_dist:.0f}px"
        pm.draw_text(label, int(best_screen.x) + 8, int(best_screen.y) - 8,
                     10, pm.get_color("red"))


plugin = FOVIndicatorPlugin()
```

---

## Example 6 — Notification System

Draws timed pop-up messages on screen. Other plugins can call `notify()` to
show messages. Shows how to share state between plugins.

```python
import pyMeow as pm
import time
from core.plugin import Plugin

# Module-level list — survives across frames
_notifications: list = []


def notify(text: str, duration: float = 3.0, color: str = "white"):
    """
    Show a message on screen for `duration` seconds.
    Import and call this from any other plugin:

        from features.notifications import notify
        notify("Teleported!", duration=2.0, color="green")
    """
    _notifications.append({
        "text":    text,
        "color":   color,
        "expires": time.time() + duration,
    })
    # Cap list size
    if len(_notifications) > 15:
        _notifications.pop(0)


class NotificationPlugin(Plugin):
    PLUGIN_NAME = "Notifications"
    PLUGIN_TAB  = "Visuals"

    def register(self, builder, mem, settings):
        builder.add_card("Notifications")
        builder.add_toggle("Show Notifications", "notif_enabled", default=True)
        builder.add_divider()
        builder.add_dropdown("Position", "notif_pos",
                             options=["Top Left", "Top Right",
                                      "Bottom Left", "Bottom Right"],
                             default="Top Left")
        builder.add_divider()
        builder.add_button("Send Test Message", self._test)
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("notif_enabled", True):
            return

        now    = time.time()
        active = [n for n in _notifications if n["expires"] > now]
        _notifications.clear()
        _notifications.extend(active)

        if not active:
            return

        vp       = mem.get_viewport()
        pos      = settings.get("notif_pos", "Top Left")
        pad      = 12
        line_h   = 20

        # Pick a corner
        if pos == "Top Left":
            x = pad;                  y = pad
        elif pos == "Top Right":
            x = int(vp.x) - 200;     y = pad
        elif pos == "Bottom Left":
            x = pad;                  y = int(vp.y) - len(active) * line_h - pad
        else:
            x = int(vp.x) - 200;     y = int(vp.y) - len(active) * line_h - pad

        for i, notif in enumerate(active):
            text  = notif["text"]
            box_w = len(text) * 7 + 10

            # Dark background for readability
            pm.draw_rectangle(x - 4, y + i * line_h - 2, box_w, 16,
                               pm.get_color("darkgray"))
            pm.draw_text(text, x, y + i * line_h, 12,
                         pm.get_color(notif["color"]))

    def _test(self):
        notify("Test notification — white",  duration=3.0, color="white")
        notify("Test notification — green",  duration=4.0, color="green")
        notify("Test notification — yellow", duration=5.0, color="yellow")
        notify("Test notification — red",    duration=6.0, color="red")


plugin = NotificationPlugin()
```

Using notifications from another plugin:
```python
from features.notifications import notify

notify("Teleported successfully!", duration=2.0, color="green")
notify("Target not found",         duration=3.0, color="red")
```

---

## Example 7 — Background Thread

Shows the correct pattern for doing slow / looping work without blocking the overlay.
The thread checks a stop event so it exits cleanly when the feature is disabled.

```python
import threading
import time
from core.plugin import Plugin
from offsets     import OFFSETS


class BackgroundWorkerPlugin(Plugin):
    PLUGIN_NAME = "Background Worker"
    PLUGIN_TAB  = "Misc"

    def __init__(self):
        # Always initialise thread state in __init__
        self._thread   = None
        self._stop_evt = threading.Event()

    def register(self, builder, mem, settings):
        builder.add_card("Background Worker Example")
        builder.add_toggle(
            "Enable",
            "bgwork_enabled",
            default=False,
            # When user turns it OFF, signal the thread to stop immediately
            on_change=lambda v: self._stop_evt.set() if not v else None
        )
        builder.add_divider()
        builder.add_slider("Interval", "bgwork_interval",
                           from_=0.1, to=5.0, fmt="{:.1f}s", default=1.0)
        builder.add_label("Runs the worker once per interval")
        builder.add_space()

    def draw(self, mem, settings):
        # draw() runs every frame — use it to start/stop the thread
        if settings.get("bgwork_enabled", False):
            if self._thread is None or not self._thread.is_alive():
                # Clear the stop signal and launch a fresh thread
                self._stop_evt.clear()
                self._thread = threading.Thread(
                    target=self._loop,
                    args=(mem, settings),
                    daemon=True    # daemon=True means it dies when the app closes
                )
                self._thread.start()
        else:
            self._stop_evt.set()

    def on_stop(self, mem):
        # Stop the thread when overlay closes
        self._stop_evt.set()

    def _loop(self, mem, settings):
        """
        Runs in a background thread.
        Use this for anything that needs to sleep, poll slowly,
        or do work that would lag the overlay if run every frame.
        """
        while not self._stop_evt.is_set():

            # Always check if the overlay is still running
            if not settings.get("esp_enabled"):
                break
            if not settings.get("bgwork_enabled"):
                break

            try:
                # ── Do your background work here ──────────────────────────
                # Example: read and print our own health every interval
                lp = getattr(mem, "local_player", None)
                if lp:
                    char = mem.read_ptr(lp + int(OFFSETS["ModelInstance"], 16))
                    if char:
                        hum = mem.find_class(char, "Humanoid")
                        if hum:
                            hp = mem.read_float(hum + int(OFFSETS["Health"], 16))
                            print(f"[bg] your health = {hp:.0f}")

            except Exception:
                pass

            # wait() is better than sleep() because it can be interrupted
            # immediately when stop_evt is set — no waiting for the full interval
            self._stop_evt.wait(timeout=settings.get("bgwork_interval", 1.0))


plugin = BackgroundWorkerPlugin()
```

---

## Example 8 — Dropdown Controlling Draw Behaviour

Shows how a dropdown changes what happens at runtime without any extra toggles.

```python
import pyMeow as pm
import math
from core.plugin import Plugin
from mem         import vec3


class SnaplineStylePlugin(Plugin):
    PLUGIN_NAME = "Snapline Style"
    PLUGIN_TAB  = "Visuals"

    def register(self, builder, mem, settings):
        builder.add_card("Snaplines")
        builder.add_toggle("Enable", "snap_enabled", default=False)
        builder.add_divider()

        # First dropdown controls where the LINE STARTS
        builder.add_dropdown(
            "Origin",
            "snap_origin",
            options=["Bottom Center", "Top Center", "Crosshair", "Left Edge", "Right Edge"],
            default="Bottom Center"
        )

        builder.add_divider()

        # Second dropdown controls where the LINE ENDS on each player
        builder.add_dropdown(
            "Target",
            "snap_target",
            options=["Feet", "Head", "Center"],
            default="Feet"
        )
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("snap_enabled", False):
            return

        vp     = mem.get_viewport()
        origin = settings.get("snap_origin", "Bottom Center")
        target = settings.get("snap_target", "Feet")

        # Compute start point from dropdown
        if origin == "Bottom Center":
            sx, sy = int(vp.x / 2), int(vp.y)
        elif origin == "Top Center":
            sx, sy = int(vp.x / 2), 0
        elif origin == "Crosshair":
            sx, sy = int(vp.x / 2), int(vp.y / 2)
        elif origin == "Left Edge":
            sx, sy = 0, int(vp.y / 2)
        else:  # Right Edge
            sx, sy = int(vp.x), int(vp.y / 2)

        for p in mem.get_players():
            # Compute end point from dropdown
            if target == "Head":
                pos = p["head_pos"]
            elif target == "Center":
                pos = p["root_pos"]
            else:  # Feet
                r = p["root_pos"]
                s = p["player_size"]
                pos = vec3(r.x, r.y - s.y / 2, r.z)

            screen = mem.world_to_screen(pos)
            if screen.x == -1:
                continue

            pm.draw_line(sx, sy, int(screen.x), int(screen.y), pm.get_color("white"))


plugin = SnaplineStylePlugin()
```

---

## Example 9 — Tool Window (Sidebar Button → Popup)

Opens a popup window from the TOOLS section in the sidebar.
Shows how to build a scrollable list inside a CTkToplevel.

```python
import threading
import customtkinter as ctk
from core.plugin     import Plugin
from core.ui_builder import (F, BG_BASE, BG_SIDEBAR, BG_CARD, BG_CARD2,
                              BG_BTN, BG_BTN_HOV, BORDER,
                              TEXT_WHITE, TEXT_DARK, GREEN, Divider)
from offsets import OFFSETS


class PlayerListWindow(ctk.CTkToplevel):
    def __init__(self, mem):
        super().__init__()
        self.mem  = mem
        self._job = None

        self.title("ironware — players")
        self.geometry("360x440")
        self.configure(fg_color=BG_BASE)
        self.protocol("WM_DELETE_WINDOW", self._close)

        self._build()
        self._refresh()
        self._schedule()   # auto-refresh every 5 seconds

    def _build(self):
        # Top bar
        top = ctk.CTkFrame(self, fg_color=BG_SIDEBAR, corner_radius=0, height=42)
        top.pack(fill="x"); top.pack_propagate(False)
        ctk.CTkLabel(top, text="PLAYERS IN SERVER",
                     font=F(10, "bold"), text_color=TEXT_DARK).pack(side="left", padx=14)
        ctk.CTkButton(top, text="↺", width=34, height=26,
                      fg_color=BG_BTN, hover_color=BG_BTN_HOV,
                      text_color=TEXT_WHITE, border_color=BORDER, border_width=1,
                      font=F(12, "bold"), corner_radius=4,
                      command=self._refresh).pack(side="right", padx=10, pady=8)

        self._status = ctk.CTkLabel(self, text="", font=F(9), text_color=TEXT_DARK)
        self._status.pack(anchor="w", padx=14, pady=(4, 2))

        outer = ctk.CTkFrame(self, fg_color=BG_CARD, corner_radius=8,
                             border_width=1, border_color=BORDER)
        outer.pack(fill="both", expand=True, padx=12, pady=(0, 12))

        self._scroll = ctk.CTkScrollableFrame(
            outer, fg_color="transparent",
            scrollbar_button_color=BG_BTN,
            scrollbar_button_hover_color=BG_BTN_HOV)
        self._scroll.pack(fill="both", expand=True, padx=4, pady=4)

    def _refresh(self):
        """Fetch player data in a background thread so the UI doesn't freeze."""
        def _w():
            players = []
            try:
                lp = getattr(self.mem, "local_player", None)
                ps = getattr(self.mem, "players", None)
                if ps:
                    for ptr in self.mem.get_children(ps):
                        if not ptr:
                            continue
                        name = self.mem.get_name(ptr)
                        if not name:
                            continue
                        is_you = ptr == lp

                        # Read health
                        hp = None
                        try:
                            char = self.mem.read_ptr(ptr + int(OFFSETS["ModelInstance"], 16))
                            if char:
                                hum = self.mem.find_class(char, "Humanoid")
                                if hum:
                                    hp = self.mem.read_float(hum + int(OFFSETS["Health"], 16))
                        except Exception:
                            pass

                        players.append({"name": name, "you": is_you, "hp": hp})

            except Exception:
                pass

            # Always update UI from the main thread
            self.after(0, lambda: self._populate(players))

        threading.Thread(target=_w, daemon=True).start()

    def _populate(self, players):
        for w in self._scroll.winfo_children():
            w.destroy()

        if not players:
            ctk.CTkLabel(self._scroll, text="No players found.",
                         font=F(11), text_color=TEXT_DARK).pack(pady=20)
            self._set_status("No players")
            return

        self._set_status(f"{len(players)} player{'s' if len(players) != 1 else ''} in server")

        for p in players:
            row = ctk.CTkFrame(self._scroll, fg_color=BG_CARD2, corner_radius=6,
                               border_width=1, border_color=BORDER)
            row.pack(fill="x", pady=2)

            name_color = GREEN if p["you"] else TEXT_WHITE
            name_text  = p["name"] + ("  (you)" if p["you"] else "")
            ctk.CTkLabel(row, text=name_text, font=F(11, "bold"),
                         text_color=name_color, anchor="w"
                         ).pack(side="left", padx=12, pady=8)

            if p["hp"] is not None:
                ctk.CTkLabel(row, text=f"{p['hp']:.0f} hp", font=F(10),
                             text_color=TEXT_DARK
                             ).pack(side="right", padx=12)

    def _set_status(self, msg):
        try:
            self._status.configure(text=msg)
        except Exception:
            pass

    def _schedule(self):
        try:
            self._job = self.after(5000, lambda: (self._refresh(), self._schedule()))
        except Exception:
            pass

    def _close(self):
        try:
            if self._job:
                self.after_cancel(self._job)
        except Exception:
            pass
        self.destroy()


class PlayerListPlugin(Plugin):
    PLUGIN_NAME        = "Player List"
    PLUGIN_TAB         = None                  # no tab UI
    PLUGIN_TOOL_BUTTON = "👥   Player List"    # sidebar button

    def __init__(self):
        self._win = None

    def open_tool(self, mem):
        # Only open one window at a time
        if self._win:
            try:
                self._win.lift(); self._win.focus(); return
            except Exception:
                self._win = None

        self._win = PlayerListWindow(mem)

        def _on_close():
            self._win._close()
            self._win = None

        self._win.protocol("WM_DELETE_WINDOW", _on_close)


plugin = PlayerListPlugin()
```

---

## Example 10 — Reading Instance Properties

Shows how to navigate the instance tree to find specific objects and read
their data. Useful for building game-specific features.

```python
import pyMeow as pm
from core.plugin import Plugin
from offsets     import OFFSETS


class InstanceReaderPlugin(Plugin):
    PLUGIN_NAME = "Instance Reader"
    PLUGIN_TAB  = "Misc"

    def register(self, builder, mem, settings):
        builder.add_card("Instance Reader")
        builder.add_toggle("Enable", "ireader_enabled", default=False)
        builder.add_label("Prints tool names and scans workspace to console")
        builder.add_space()

    def draw(self, mem, settings):
        if not settings.get("ireader_enabled", False):
            return

        # ── Scan all players for held tools ───────────────────────────────────
        lp = getattr(mem, "local_player", None)
        ps = getattr(mem, "players", None)
        if ps:
            for player_ptr in mem.get_children(ps):
                if not player_ptr or player_ptr == lp:
                    continue
                try:
                    name = mem.get_name(player_ptr)
                    char = mem.read_ptr(player_ptr + int(OFFSETS["ModelInstance"], 16))
                    if not char or mem.get_class(char) != "Model":
                        continue

                    # Iterate character children
                    for child in mem.get_children(char):
                        cls = mem.get_class(child)

                        if cls == "Tool":
                            # Player is holding this tool right now
                            tool_name = mem.get_name(child)
                            # Use tool_name however you want —
                            # highlight them, track it, show in ESP, etc.
                            _ = (name, tool_name)

                        elif cls == "Humanoid":
                            hp  = mem.read_float(child + int(OFFSETS["Health"],    16))
                            mhp = mem.read_float(child + int(OFFSETS["MaxHealth"], 16))
                            _ = (hp, mhp)

                except Exception:
                    continue

        # ── Find a specific named part in Workspace ───────────────────────────
        ws = getattr(mem, "workspace", None)
        if ws:
            # Look for a part named "Checkpoint" anywhere in Workspace
            checkpoint = mem.find_child(ws, "Checkpoint")
            if checkpoint:
                pos = mem.read_pos(checkpoint)
                if pos:
                    screen = mem.world_to_screen(pos)
                    if screen.x != -1:
                        pm.draw_circle(int(screen.x), int(screen.y), 10,
                                       pm.get_color("yellow"))
                        pm.draw_text("Checkpoint",
                                     int(screen.x) - 35, int(screen.y) - 22,
                                     11, pm.get_color("yellow"))

            # Find all parts with class "SpawnLocation"
            for child in mem.get_children(ws):
                if mem.get_class(child) == "SpawnLocation":
                    pos = mem.read_pos(child)
                    if pos:
                        screen = mem.world_to_screen(pos)
                        if screen.x != -1:
                            pm.draw_circle_lines(int(screen.x), int(screen.y), 8,
                                                  pm.get_color("green"))


plugin = InstanceReaderPlugin()
```

---

## Common Patterns Cheatsheet

### Get local player's Humanoid
```python
from offsets import OFFSETS
lp   = getattr(mem, "local_player", None)
char = mem.read_ptr(lp + int(OFFSETS["ModelInstance"], 16))
hum  = mem.find_class(char, "Humanoid")
```

### Write a float every frame (survives respawn)
```python
def draw(self, mem, settings):
    if settings.get("my_toggle"):
        h = get_humanoid(mem)
        if h:
            mem.write_float(h + int(OFFSETS["WalkSpeed"], 16), 100.0)
```

### World position → screen + guard off-screen
```python
screen = mem.world_to_screen(p["head_pos"])
if screen.x == -1:
    continue   # off screen — always check this
pm.draw_circle(int(screen.x), int(screen.y), 5, pm.get_color("red"))
```

### Find any named child
```python
tool = mem.find_child(character_ptr, "Sword")     # by name
hum  = mem.find_class(character_ptr, "Humanoid")  # by class name
```

### Write raw bytes (velocity / CFrame)
```python
import struct
# velocity: 3 floats = 12 bytes
mem.write(prim + int(OFFSETS["Velocity"], 16), struct.pack("fff", vx, vy, vz))

# identity CFrame at position x,y,z: 12 floats = 48 bytes
cf = struct.pack("ffffffffffff",
     1,0,0, 0,1,0, 0,0,1, x, y, z)
mem.write(prim + cf_offset, cf)
```

### Screen center
```python
vp = mem.get_viewport()
cx = int(vp.x / 2)
cy = int(vp.y / 2)
```

### Reset toggle from code
```python
handle = builder.add_toggle("Enable", "my_key", default=False)
# in a button callback:
handle.set(False)   # updates both the widget AND settings["my_key"]
```

### Start a background thread safely
```python
def draw(self, mem, settings):
    if settings.get("my_key"):
        if self._thread is None or not self._thread.is_alive():
            self._stop_evt.clear()
            self._thread = threading.Thread(
                target=self._loop, args=(mem, settings), daemon=True)
            self._thread.start()
    else:
        self._stop_evt.set()
```

---

## Onyx Theme Colors

```python
from core.ui_builder import (
    BG_BASE,      # "#080808"  main background
    BG_SIDEBAR,   # "#0f0f0f"  sidebar
    BG_CARD,      # "#141414"  card
    BG_CARD2,     # "#1a1a1a"  selected / hover
    BG_INPUT,     # "#1e1e1e"  text fields
    BG_BTN,       # "#1e1e1e"  button bg
    BG_BTN_HOV,   # "#2a2a2a"  button hover
    BORDER,       # "#2a2a2a"  borders
    TEXT_WHITE,   # "#ffffff"
    TEXT_GREY,    # "#999999"
    TEXT_DARK,    # "#555555"  muted hints
    GREEN,        # "#4ade80"
    RED_SOFT,     # "#f87171"
    RED_BTN,      # "#7f1d1d"  danger button
    RED_BTN_HOV,  # "#991b1b"
    F,            # font helper — F(12) or F(12, "bold")
    Divider,      # 1px separator widget
)
```

---

## Plugin File Checklist

- [ ] File saved in `features/`
- [ ] Class inherits `Plugin` from `core.plugin`
- [ ] `PLUGIN_NAME` is set
- [ ] `PLUGIN_TAB` is `"Visuals"` / `"Aimbot"` / `"Misc"` / `None`
- [ ] `register()` builds UI with `builder`
- [ ] `draw()` starts with `if not settings.get("my_key"): return`
- [ ] `on_stop()` resets memory writes and signals threads to stop
- [ ] `plugin = MyClass()` at the very bottom of the file
