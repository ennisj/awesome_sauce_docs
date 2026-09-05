# BitLCD theming guide

This guide walks through creating marquee themes for BitLCD, from a
minimal JSON-only bezel to Lua-scripted procedural animation.

## Quick start

A theme is a folder on your USB drive:

```
<drive>/bitlcd/themes/my-theme/
    theme.json
    frame.png
    background.png
```

The simplest working theme:

```json
{
  "format": 2,
  "id": "my-theme",
  "scenes": {
    "selection": {
      "background": "#000000",
      "layers": [
        {
          "id": "media",
          "type": "media",
          "source": "$media",
          "rect": [0, 0, 1, 1],
          "fit": "contain"
        }
      ]
    }
  },
  "events": {
    "selection": "selection"
  }
}
```

Activate it in `<drive>/bitlcd/config.json`:

```json
{
  "version": 2,
  "theme": "my-theme"
}
```

For complete per-game control, put a manifest in a searchable content directory
and keep its media in an underscore-prefixed directory:

```text
<drive>/bitlcd/bitlcd/arcade/pacman.json
<drive>/bitlcd/bitlcd/arcade/_pacman/
    pacman.jpg
    pacman.mp4
    logo.png
    fanart-1.jpg
    fanart-2.jpg
    move-list.png
```

```json
{
  "format": 1,
  "theme": "maze-theme",
  "media": {
    "image": "_pacman/pacman.jpg",
    "video": "_pacman/pacman.mp4",
    "logo": "_pacman/logo.png",
    "fanart": [
      "_pacman/fanart-1.jpg",
      "_pacman/fanart-2.jpg"
    ],
    "move_list": "_pacman/move-list.png"
  }
}
```

Manifests are discovered recursively in `bitlcd/`, `ext/`, and `thirdparty/`.
They may exist without a same-stem JPG or MP4 in the searchable directory.
Media paths are relative to the manifest, may enter `_` directories, and must
remain inside the drive's `bitlcd/` tree. A string declares one item; an array
declares an ordered collection. Slot names use lowercase letters, digits,
dots, underscores, and hyphens.

`image` and `video` are the reserved primary slots. Other names such as `logo`,
`fanart`, and `move_list` are author-defined. If either primary slot is omitted,
legacy same-stem discovery supplies that type when possible. A theme-only
manifest beside legacy media therefore continues to work.

To share a theme, point several manifests at the same theme ID. These are
references to one theme package, not copies or inherited themes.

Games without a manifest theme use the global `theme`. If a selected theme is
missing, the global theme is tried before the normal `default`, `classic`, and
built-in fallbacks. The older central `game_themes` object is still accepted,
but a per-game file takes precedence.

Any directory whose name starts with `_` is excluded from recursive manifest
and legacy-media discovery. Its files are reachable only through explicit
manifest paths. The underscore rule applies at every nesting level.

Theme IDs use lowercase letters, digits, dots, underscores, and hyphens.
The first character must be a letter or digit.

### Theme selection order

At startup and on each game selection, marqueed resolves themes from mounted
drives in this order:

1. The game's JSON manifest, when present.
2. The global theme named in `config.json`.
3. A theme named `default`.
4. A theme named `classic`.
5. The built-in classic presentation.

## Scenes and layers

A scene has a background color and an ordered list of layers. Later
layers draw on top of earlier ones.

```json
{
  "scenes": {
    "selection": {
      "background": "#05010a",
      "layers": [
        {
          "id": "bg",
          "type": "image",
          "source": "background.png",
          "rect": [0, 0, 1, 1],
          "fit": "cover"
        },
        {
          "id": "media",
          "type": "media",
          "source": "$media",
          "rect": [0.04, 0.08, 0.92, 0.84],
          "fit": "contain"
        },
        {
          "id": "frame",
          "type": "image",
          "source": "frame.png",
          "rect": [0, 0, 1, 1]
        },
        {
          "id": "title",
          "type": "text",
          "value": "$title",
          "font": "font.ttf",
          "size": 0.09,
          "color": "#ffd85a",
          "anchor": "bottom-center"
        }
      ],
      "transition": {"type": "fade", "ms": 250}
    }
  }
}
```

### Layer types

| Type | Purpose |
|---|---|
| `color` | Filled rectangle |
| `image` | Static image from the theme folder |
| `media` | Title media resolved by marqueed (image or video) |
| `video` | Explicit video with playback control |
| `text` | Bound or literal UTF-8 text |

Every layer has an `id`, `rect`, and `opacity` (0 to 1, default 1).
Image, media, video, and text layers also support `transform` and
`timeline`.

### Geometry

`rect` is `[x, y, width, height]` in normalized coordinates. `[0, 0, 1, 1]`
covers the full panel.

The optional `rect_units` property selects the coordinate system:

| Unit | Description |
|---|---|
| `panel` | Default. x/width use panel width, y/height use panel height |
| `vh` | All values use panel height (useful for square regions) |
| `px` | Physical pixels |

Scalar sizes like `text.size` are fractions of the panel height: `0.09`
is about 32 pixels on a 360-pixel panel.

### Fitting

Image and video layers have a `fit` property:

- `contain` — scale to fit inside the rect, preserving aspect ratio
  (default).
- `cover` — scale to fill the rect, cropping the overflow.
- `stretch` — distort to fill the rect exactly.

Layers clip to their rectangle by default. Set `clip: false` to allow
overflow.

### Colors

Colors are `#RRGGBB` or `#RRGGBBAA`. The final pair is alpha (FF =
opaque). A layer's `opacity` multiplies its color or texture alpha.

## Bindings

String values starting with `$` are replaced with data from the current
presentation:

| Binding | Content |
|---|---|
| `$title` | Game title |
| `$title2` | Secondary title |
| `$romname` | ROM filename |
| `$leaderboard` | Leaderboard text |
| `$payload_type` | Numeric payload type |
| `$media` | Preferred media (image if both exist, otherwise whatever is available) |
| `$media.image` | Title image path |
| `$media.video` | Title video path |
| `$media.<slot>` | First available item in a named manifest media slot |

Bindings are complete-value replacements — `"$title"` works, but
`"Game: $title"` is treated as a literal string. Use `$$` to escape a
dollar sign: `"$$5.00"` renders as `$5.00`.

For an array-valued slot, declarative JSON uses the first available item. Lua
receives every ordered item under `ctx.media.slots.<name>` and can select among
them dynamically. Layers using optional named media should set
`"required": false`.

```json
{
  "id": "game-logo",
  "type": "image",
  "source": "$media.logo",
  "required": false,
  "rect": [0.7, 0.05, 0.25, 0.3],
  "fit": "contain"
}
```

## Text layers

```json
{
  "id": "title",
  "type": "text",
  "value": "$title",
  "rect": [0.04, 0.1, 0.92, 0.8],
  "fonts": ["custom.ttf"],
  "size": 0.09,
  "min_size": 0.045,
  "color": "#FFD700",
  "anchor": "bottom-center",
  "wrap": "word",
  "max_lines": 2,
  "overflow": "shrink"
}
```

`font` is shorthand for a single-element `fonts` array. A built-in
fallback font covers CJK characters automatically.

**Anchors:** `top-left`, `top-center`, `top-right`, `center-left`,
`center`, `center-right`, `bottom-left`, `bottom-center`, `bottom-right`.

**Wrap modes:** `none` (default), `word`, `character`.

**Overflow modes:**

- `clip` — clip at the rectangle edge (default).
- `ellipsis` — replace the end of the last visible line with an
  ellipsis.
- `shrink` — reduce the font toward `min_size`, then apply ellipsis.
- `marquee` — scroll horizontally.

## Transitions

Transitions animate between the old scene and the new scene:

```json
{
  "transition": {"type": "fade", "ms": 250}
}
```

Available types: `cut` (instant), `fade`, `slide_left`, `slide_right`.

## Events

The `events` object maps presentation events to scenes or sequences:

```json
{
  "events": {
    "boot": "boot-screen",
    "selection": "selection",
    "media_missing": "missing",
    "idle": {"scene": "idle-screen"},
    "attract": {"sequence": "attract-loop"}
  }
}
```

For selection events, you can use more specific keys. The engine checks
them in order:

1. `selection.image` — when only a title image exists.
2. `selection.video` — when only a title video exists.
3. `selection.image_video` — when both exist.
4. `selection` — fallback for any selection.

`media_missing` fires when neither image nor video is found.

### Event reference

| Event | When it fires |
|---|---|
| `boot` | After theme activation at startup |
| `selection` | Host sends a game selection |
| `media_missing` | No media found for a selection |
| `idle` | No activity for `idle.after_s` seconds |
| `attract` | Attract mode selects an item |
| `video_ended` | A finite video finishes its configured plays |
| `video_error` | Video open/decode failure |
| `timer` | A Lua-created timer expires |
| `reload` | Theme successfully reloaded |

## Timelines

Timelines animate layer properties over time. They run in C++ at the
render frame rate without involving Lua.

### Opacity timeline

Fade a title out after 2.5 seconds:

```json
{
  "id": "title",
  "type": "text",
  "value": "$title",
  "anchor": "bottom-center",
  "timeline": {
    "trigger": "scene_enter",
    "keyframes": [
      {"at_ms": 0,    "opacity": 1},
      {"at_ms": 2500, "opacity": 1},
      {"at_ms": 3000, "opacity": 0}
    ],
    "fill": "forwards"
  }
}
```

### Transform timeline

Animate position, scale, rotation, and pivot:

```json
{
  "id": "logo",
  "type": "image",
  "source": "logo.png",
  "rect": [0.35, 0.2, 0.3, 0.6],
  "fit": "contain",
  "timeline": {
    "trigger": "scene_enter",
    "loop": true,
    "keyframes": [
      {"at_ms": 0,    "transform": {"scale": [1.0, 1.0]}},
      {"at_ms": 500,  "transform": {"scale": [1.15, 1.15]}},
      {"at_ms": 1000, "transform": {"scale": [1.0, 1.0]}}
    ]
  }
}
```

Transform properties are set inside keyframe `transform` objects:

| Property | Default | Description |
|---|---|---|
| `translate` | `[0, 0]` | Offset as `[x, y]` in rect units |
| `scale` | `[1, 1]` | Scale as `[x, y]` multipliers |
| `rotation_deg` | `0` | Rotation in degrees |
| `pivot` | `[0.5, 0.5]` | Rotation/scale center, normalized within the layer |

### Independent tracks

Each property is an independent track. A keyframe only affects the
properties it mentions — unmentioned properties keep their base values
or interpolate between their own keyframes independently.

This example pulses for two seconds, then rotates for one second, and
loops:

```json
{
  "keyframes": [
    {"at_ms": 0,    "transform": {"scale": [1.0, 1.0]}},
    {"at_ms": 500,  "transform": {"scale": [1.15, 1.15]}},
    {"at_ms": 1000, "transform": {"scale": [1.0, 1.0]}},
    {"at_ms": 1500, "transform": {"scale": [1.15, 1.15]}},
    {"at_ms": 2000, "transform": {"scale": [1.0, 1.0], "rotation_deg": 0}},
    {"at_ms": 3000, "transform": {"rotation_deg": 360}}
  ]
}
```

Scale pulses from 0–2000ms. Rotation only starts at 2000ms and runs to
3000ms. They don't interfere with each other.

### Timeline properties

| Property | Values | Description |
|---|---|---|
| `trigger` | `scene_enter`, `value_change` | When to start/restart |
| `binding` | e.g. `"$title"` | Required with `value_change` — restarts when this value changes |
| `fill` | `forwards`, `reset` | What happens after the last keyframe |
| `loop` | `true`/`false` | Restart from the beginning when the last keyframe is reached |

### Static transforms

You can set a static transform on a layer without a timeline:

```json
{
  "id": "logo",
  "type": "image",
  "source": "logo.png",
  "rect": [0.4, 0.2, 0.2, 0.6],
  "transform": {
    "rotation_deg": 15,
    "scale": [1.2, 1.2]
  }
}
```

## Video playback

### Basic video

```json
{
  "id": "gameplay",
  "type": "video",
  "source": "$media.video",
  "rect": [0, 0, 1, 1],
  "fit": "contain",
  "playback": {
    "loop": true
  }
}
```

Without a `playback` descriptor, video plays once and stops. Add
`"playback": {"loop": true}` for continuous playback.

### Playback properties

| Property | Default | Description |
|---|---|---|
| `id` | layer ID | Playback instance identifier |
| `loop` | `false` | Play continuously |
| `count` | `1` | Total number of plays (1–1000, mutually exclusive with `loop`) |
| `start` | `0` | Offset in seconds, or a random-start object |
| `clip` | full file | `{"in_s": number, "out_s": number}` playback range |

### Random start

```json
{
  "playback": {
    "loop": true,
    "start": {"mode": "random", "min_s": 0}
  }
}
```

Each playback instance gets a deterministic random offset. Different
playback IDs get different offsets.

### Multiple video instances

Each distinct `playback.id` creates an independent decoder. This example
creates four independently phased copies:

```json
{
  "id": "tile-1",
  "type": "video",
  "source": "$media.video",
  "rect": [0.01, 0.1, 0.235, 0.8],
  "fit": "cover",
  "playback": {"id": "player-1", "loop": true, "start": {"mode": "random"}}
},
{
  "id": "tile-2",
  "type": "video",
  "source": "$media.video",
  "rect": [0.255, 0.1, 0.235, 0.8],
  "fit": "cover",
  "playback": {"id": "player-2", "loop": true, "start": {"mode": "random"}}
}
```

Declare the number of instances you need:

```json
{
  "requires": {"video_instances": 4}
}
```

Themes requiring more instances than the device supports are rejected at
activation.

### Shared playback

To show the same decoded frame in multiple places (same instant, one
decoder), reuse the same `playback.id`:

```json
{
  "id": "copy-1", "type": "video", "source": "$media.video",
  "rect": [0.00, 0, 0.25, 1], "fit": "cover",
  "playback": {"id": "shared", "loop": true}
},
{
  "id": "copy-2", "type": "video", "source": "$media.video",
  "rect": [0.25, 0, 0.25, 1], "fit": "cover",
  "playback": {"id": "shared", "loop": true}
}
```

This consumes only one playback instance regardless of how many layers
share the ID.

## Sequences

Sequences chain scenes together. Map an event to a sequence instead of a
scene:

```json
{
  "events": {
    "selection": {"sequence": "intro-then-still"}
  }
}
```

### Timed steps

Cycle images every five seconds:

```json
{
  "sequences": {
    "slideshow": {
      "loop": true,
      "steps": [
        {"scene": "slide-1", "duration_ms": 5000},
        {"scene": "slide-2", "duration_ms": 5000},
        {"scene": "slide-3", "duration_ms": 5000}
      ]
    }
  }
}
```

### Video-driven steps

Advance when a video finishes:

```json
{
  "sequences": {
    "intro-then-still": {
      "steps": [
        {
          "scene": "intro",
          "until": [
            {"event": "video_ended", "playback_id": "intro-player"},
            {"event": "video_error", "playback_id": "intro-player"}
          ]
        },
        {"scene": "still"}
      ]
    }
  }
}
```

`until` accepts one matcher or an array of alternatives — the first
match advances the step. A step with no `duration_ms` or `until` is
terminal.

### Counted playback in sequences

Override a video layer's play count for one sequence step:

```json
{
  "scene": "media-video",
  "until": [
    {"event": "video_played", "playback_id": "media", "count": 3},
    {"event": "video_error", "playback_id": "media"}
  ]
}
```

The layer's own `loop: true` is overridden — the player runs exactly
three times for this step. This lets the same scene loop continuously
when used directly and play a bounded number of times in a sequence.

## Idle and attract mode

```json
{
  "idle": {
    "after_s": 60,
    "target": {"scene": "idle-screen"}
  },
  "attract": {
    "enabled": true,
    "order": "shuffle",
    "dwell_s": 20,
    "sources": ["rom", "drive"]
  }
}
```

`idle.after_s` is the seconds of inactivity before the idle scene
activates. Set to `0` to disable. `idle.media` is a shorthand for a
full-panel looping video: `"media": "idle.mp4"`.

Attract mode cycles through content from ROM and/or drive sources.
`order` is `sequential` or `shuffle`.

## Inheritance

Extend an existing theme instead of copying it entirely:

```json
{
  "format": 2,
  "id": "neon-variant",
  "extends": "neon",
  "scenes": {
    "selection": {
      "background": "#1a0033",
      "layers": [
        {
          "id": "title",
          "color": "#ff44aa"
        }
      ]
    }
  }
}
```

Scenes merge by name. Layers merge by `id` — a matching layer replaces
only the fields you specify, keeping its draw order position.

### Adding layers

New layers append after inherited ones. Use `before` or `after` to
control position:

```json
{
  "id": "badge",
  "type": "image",
  "source": "badge.png",
  "rect": [0.9, 0.02, 0.08, 0.2],
  "after": "frame"
}
```

### Removing layers

```json
{"id": "title", "remove": true}
```

## Lua scripting

For themes that need conditional logic, state, timers, or procedural
animation, add a `theme.lua` script.

Reference it in the manifest:

```json
{
  "format": 2,
  "id": "dynamic-theme",
  "script": "theme.lua"
}
```

### Event hook

The script returns a table with an `on_event` function:

```lua
return {
  on_event = function(ctx)
    if not ctx.media.found then
      return "missing"
    end

    return {
      scene = "selection",
      vars = {
        title = ctx.title,
        accent = ctx.payload_type == 2 and "#42e8ff" or "#ffd85a"
      }
    }
  end
}
```

`on_event` receives a context table and can return:

- `nil` — keep the current scene.
- A string — select a named scene from the manifest.
- A table with `scene` (string) — select a named scene, optionally with
  `vars` to override bound values.
- A table with `scene` (table) — an inline scene definition.
- A table with `sequence` (string) — start a named sequence.

### Context fields

The context table passed to `on_event`:

| Field | Type | Description |
|---|---|---|
| `ctx.event` | string | Event name (`selection`, `boot`, `timer`, etc.) |
| `ctx.sequence` | number | Presentation sequence number |
| `ctx.romname` | string | ROM filename |
| `ctx.title` | string | Game title |
| `ctx.title2` | string | Secondary title |
| `ctx.leaderboard` | string | Leaderboard text |
| `ctx.payload_type` | number | Payload type identifier |
| `ctx.elapsed_ms` | number | Milliseconds since Lua state creation |
| `ctx.media.found` | boolean | Whether primary image or video media was found |
| `ctx.media.kind` | string | `image`, `video`, `image_video`, or `not_found` |
| `ctx.media.image` | string | Title image path |
| `ctx.media.video` | string | Title video path |
| `ctx.media.images` | array | Ordered primary-image collection |
| `ctx.media.videos` | array | Ordered primary-video collection |
| `ctx.media.slots.<name>` | array | Ordered items from any manifest media slot |
| `ctx.panel.width` | number | Panel width in pixels |
| `ctx.panel.height` | number | Panel height in pixels |
| `ctx.capabilities.video_instances` | number | Device video instance limit |
| `ctx.event_data` | table | Event-specific data (timer_id, playback_id, etc.) |
| `ctx.state` | table | Mutable state preserved between callbacks |

### Inline scenes

Return a complete scene from Lua:

```lua
return {
  scene = {
    background = "#050505",
    layers = {
      {
        id = "slide",
        type = "image",
        source = images[current],
        rect = {0, 0, 1, 1},
        fit = "cover"
      }
    },
    transition = {type = "fade", ms = 200}
  }
}
```

### Timers

Request a callback after a delay:

```lua
return {
  scene = "selection",
  timers = {
    {id = "next-slide", after_ms = 5000, replace = true}
  }
}
```

When the timer fires, `on_event` is called with `ctx.event == "timer"`
and `ctx.event_data.timer_id == "next-slide"`. Timers are cancelled
when a new host selection arrives.

`replace = true` replaces an existing timer with the same ID instead of
creating a duplicate.

### Persistent state

`ctx.state` is a mutable table that persists between callbacks within
the same theme activation:

```lua
on_event = function(ctx)
  if ctx.event == "selection" then
    ctx.state.count = (ctx.state.count or 0) + 1
  end
  -- ctx.state.count persists across selections
end
```

State is discarded when the theme is reloaded.

### Available libraries

Lua scripts have access to:

- `math`, `string`, `table`, `utf8` (safe portions)
- Scene and color constructors

The following are **not available** for safety: `io`, `os`, `package`,
`debug`, `dofile`, `loadfile`, `load`, `require`.

Scripts run on a dedicated worker thread with memory and instruction
limits. A script that exceeds its budget falls back to the manifest's
declarative event mappings.

## Procedural animators

For smooth, physics-based motion that can't be expressed as keyframes,
declare a procedural animator in the scene and implement it in Lua.

### Declaring an animator

In the scene's `animations` array:

```json
{
  "scenes": {
    "selection": {
      "layers": [
        {
          "id": "logo",
          "type": "image",
          "source": "logo.png",
          "rect": [0.35, 0.2, 0.3, 0.6],
          "fit": "contain"
        }
      ],
      "animations": [
        {
          "id": "logo-bounce",
          "hook": "logo_bounce",
          "tick_hz": 60,
          "targets": {
            "logo": ["translation", "scale", "rotation", "pivot", "opacity"]
          },
          "params": {
            "spring_k": 200,
            "damping": 0.8
          }
        }
      ]
    }
  }
}
```

| Field | Description |
|---|---|
| `id` | Unique identifier for this animator instance |
| `hook` | Name of the Lua animation table to call |
| `tick_hz` | Simulation rate, 1–120 (default 60) |
| `targets` | Map of layer ID to writable properties |
| `params` | Immutable numeric parameters passed to `init` |
| `composition` | Set to `"additive"` to compose with a timeline on the same layer |

### Implementing the hook

In `theme.lua`, the script returns an `animations` table alongside
`on_event`:

```lua
local animations = {}

animations.logo_bounce = {
  init = function(ctx, params)
    -- Return initial state
    return {
      y = 0,
      velocity = 0,
      spring_k = params.spring_k or 200,
      damping = params.damping or 0.8
    }
  end,

  update = function(state, frame, out)
    local dt = frame.dt_s

    -- Spring physics
    local force = -state.spring_k * state.y
    state.velocity = state.velocity + force * dt
    state.velocity = state.velocity * math.exp(-state.damping * dt)
    state.y = state.y + state.velocity * dt

    -- Write the pose
    -- args: layer, tx, ty, sx, sy, rotation_deg, pivot_x, pivot_y, opacity
    out:set_transform("logo", 0, state.y, 1, 1, 0, 0.5, 0.5, 1.0)

    -- Return true to keep ticking, false to go dormant
    return true
  end,

  on_signal = function(state, signal)
    if signal.name == "impulse" then
      state.velocity = state.velocity + signal.value
    end
  end,

  destroy = function(state, reason)
    -- Optional cleanup
  end
}

return {
  on_event = function(ctx)
    return "selection"
  end,
  animations = animations
}
```

### Animator lifecycle

1. **init** — called once when the scene is ready. Receives the
   presentation context and the `params` from the declaration. Returns
   the animator's private state table.

2. **update** — called at the declared `tick_hz` rate. Receives:
   - `state` — the mutable state table from `init`.
   - `frame.dt_s` — fixed timestep (always `1 / tick_hz`).
   - `frame.elapsed_s` — total simulation time for this animator.
   - `frame.tick` — monotonic tick counter.
   - `out` — the pose output object.

   Returns `true` to continue ticking, `false` to go dormant.

3. **on_signal** — called when an event hook sends a signal to this
   animator. A dormant animator wakes up when it receives a signal.

4. **destroy** — called when the scene, presentation, or theme is
   replaced. Optional.

### Pose output

The `out` object in `update` has one method:

```lua
out:set_transform(layer_id, tx, ty, sx, sy, rotation_deg, pivot_x, pivot_y, opacity)
```

All values are numbers. Translation is in rect units, scale is a
multiplier, rotation is degrees, pivot is normalized within the layer,
and opacity is 0 to 1.

An animator can only write to layers and properties listed in its
`targets` declaration.

### Sending signals to animators

From `on_event`, send signals to active animators:

```lua
return {
  scene = "selection",
  animation_signals = {
    {id = "logo-bounce", name = "impulse", value = 0.4}
  }
}
```

Signals are tagged with the current presentation — stale signals for
replaced scenes are discarded.

### Composition with timelines

By default, a timeline and a Lua animator cannot write the same property
on the same layer. To allow both, set `composition: "additive"` on the
animator declaration. The composition order is:

1. Base layer transform
2. Declarative timeline result
3. Lua animator result

Translation and rotation add. Scale and opacity multiply.

## Animation limits

Themes that use procedural animation must declare their maximum demand:

```json
{
  "limits": {
    "animators": 2,
    "animated_layers": 16,
    "animation_commands_per_tick": 32,
    "animation_tick_hz": 60
  }
}
```

The device profile caps each value. If an animator exceeds its budget,
it keeps its last valid pose and is disabled after repeated failures.

## Cookbook

### Bezel with fading title

```json
{
  "format": 2,
  "id": "bezel",
  "scenes": {
    "selection": {
      "background": "#0a0a0a",
      "layers": [
        {
          "id": "media",
          "type": "media",
          "source": "$media",
          "rect": [0.04, 0.06, 0.92, 0.82],
          "fit": "contain"
        },
        {
          "id": "frame",
          "type": "image",
          "source": "frame.png",
          "rect": [0, 0, 1, 1]
        },
        {
          "id": "title",
          "type": "text",
          "value": "$title",
          "font": "font.ttf",
          "size": 0.07,
          "color": "#ffffff",
          "anchor": "bottom-center",
          "rect": [0.05, 0, 0.9, 0.96],
          "timeline": {
            "trigger": "scene_enter",
            "keyframes": [
              {"at_ms": 0, "opacity": 0},
              {"at_ms": 300, "opacity": 1},
              {"at_ms": 3000, "opacity": 1},
              {"at_ms": 3500, "opacity": 0}
            ],
            "fill": "forwards"
          }
        }
      ],
      "transition": {"type": "fade", "ms": 250}
    },
    "missing": {
      "background": "#0a0a0a",
      "layers": [
        {"id": "frame", "type": "image", "source": "frame.png", "rect": [0, 0, 1, 1]},
        {
          "id": "title",
          "type": "text",
          "value": "$title",
          "size": 0.14,
          "color": "#ffffff",
          "anchor": "center"
        }
      ]
    }
  },
  "events": {
    "selection": "selection",
    "media_missing": "missing"
  }
}
```

### Image/video cycle (classic behavior)

Show the title image for three seconds, play the video once, and repeat:

```json
{
  "format": 2,
  "id": "classic-cycle",
  "scenes": {
    "image-only": {
      "layers": [
        {"id": "img", "type": "image", "source": "$media.image", "rect": [0, 0, 1, 1], "fit": "contain"}
      ]
    },
    "video-only": {
      "layers": [
        {"id": "vid", "type": "video", "source": "$media.video", "rect": [0, 0, 1, 1], "fit": "contain", "playback": {"loop": true}}
      ]
    },
    "image-scene": {
      "layers": [
        {"id": "img", "type": "image", "source": "$media.image", "rect": [0, 0, 1, 1], "fit": "contain"}
      ]
    },
    "video-scene": {
      "layers": [
        {"id": "vid", "type": "video", "source": "$media.video", "rect": [0, 0, 1, 1], "fit": "contain", "playback": {"id": "media", "loop": true}}
      ]
    }
  },
  "sequences": {
    "image-video-cycle": {
      "loop": true,
      "steps": [
        {"scene": "image-scene", "duration_ms": 3000},
        {
          "scene": "video-scene",
          "until": [
            {"event": "video_played", "playback_id": "media", "count": 1},
            {"event": "video_error", "playback_id": "media"}
          ]
        }
      ]
    }
  },
  "events": {
    "selection.image": "image-only",
    "selection.video": "video-only",
    "selection.image_video": {"sequence": "image-video-cycle"}
  }
}
```

### Pulsing logo with Lua

```json
{
  "format": 2,
  "id": "pulse-demo",
  "script": "theme.lua",
  "scenes": {
    "selection": {
      "background": "#000000",
      "layers": [
        {"id": "media", "type": "media", "source": "$media", "rect": [0, 0, 1, 1], "fit": "contain"},
        {"id": "logo", "type": "image", "source": "logo.png", "rect": [0.42, 0.3, 0.16, 0.4], "fit": "contain"}
      ],
      "animations": [
        {
          "id": "pulse",
          "hook": "pulse",
          "tick_hz": 30,
          "targets": {"logo": ["scale", "opacity"]}
        }
      ]
    }
  },
  "events": {"selection": "selection"},
  "limits": {"animators": 1, "animated_layers": 1}
}
```

`theme.lua`:

```lua
local animations = {}

animations.pulse = {
  init = function(ctx, params)
    return {t = 0}
  end,
  update = function(state, frame, out)
    state.t = state.t + frame.dt_s
    local s = 1.0 + 0.1 * math.sin(state.t * 2 * math.pi)
    local a = 0.7 + 0.3 * math.sin(state.t * 2 * math.pi)
    out:set_transform("logo", 0, 0, s, s, 0, 0.5, 0.5, a)
    return true
  end
}

return {
  on_event = function(ctx)
    return "selection"
  end,
  animations = animations
}
```

## Thread model

Understanding the thread model helps when debugging or writing advanced
themes:

- **Render thread** — owns SDL. Evaluates declarative timelines,
  applies poses, draws layers. Never runs Lua.
- **Presentation worker** — resolves events, prepares scenes, decodes
  images.
- **Theme script worker** — owns the Lua state. Runs `on_event` and
  animator `update` calls. Publishes pose snapshots that the render
  thread picks up without waiting.

The render thread never blocks on Lua. If a Lua animator runs slowly,
the render thread simply repeats the last valid pose.
