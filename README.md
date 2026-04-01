# LED Effects for Klipper

## Description

This is the standalone repository of the Klipper LED Effects module developed by [Paul McGowan](https://github.com/mental405) with contributions from myself.
It allows Klipper to run effects and animations on addressable LEDs, such as Neopixels, WS2812 or SK6812.

Check out Paul's printer:

[![Hello Friend. -International Edition-](https://i3.ytimg.com/vi/-VpZTDSu1-8/hqdefault.jpg)](https://www.youtube.com/watch?v=-VpZTDSu1-8)

See the chapter in this video from Vector3D what it can do and how to set it up (start from the beginning to learn how to connect the LEDs):

[![The 3D Printer Upgrade You didn’t know you needed](https://i3.ytimg.com/vi/14LC8Tcd_JQ/hqdefault.jpg)](https://youtu.be/14LC8Tcd_JQ?t=779)

And this one (in french) from Tom's Basement:

[![Klipper LED EFFECTS : C'est Noël avant l'heure dans votre imprimante 3D ! (Tuto Leds)](http://i3.ytimg.com/vi/6rGjlBjFhss/hqdefault.jpg)](https://www.youtube.com/watch?v=6rGjlBjFhss)

## Disclaimer
**This is work in progress and currently in "beta" state.**
I don't take any responsibility for any damage that happens while using this software.

## Support

For questions and support use the Q&A section on the [Discussions](https://github.com/julianschill/klipper-led_effect/discussions) page.

If you found a bug or you want to file a feature request open an [issue](https://github.com/julianschill/klipper-led_effect/issues).

If you need direct support or want to help by testing or contributing, please contact me on the [Klipper](https://discord.klipper3d.org/) or [Voron](https://discord.gg/voron) Discord. User: 5hagbard23

## Installation

### Automatic installation

The module can be installed into a existing Klipper installation with an install script. 

    cd ~
    git clone https://github.com/julianschill/klipper-led_effect.git
    cd klipper-led_effect
    ./install-led_effect.sh

If your directory structure differs from the usual setup you can configure the
installation script with parameters:

    ./install-led_effect.sh [-k <klipper path>] [-s <klipper service name>] [-c <configuration path>]

### Manual installation
Clone the repository:

    cd ~
    git clone https://github.com/julianschill/klipper-led_effect.git

Stop Klipper:

    systemctl stop klipper

Link the file in the Klipper directory (adjust the paths as needed):

    ln -s klipper-led_effect/src/led_effect.py ~/klipper/klippy/extras/led_effect.py

Start Klipper:

    systemctl start klipper

Add the updater section to moonraker.conf and restart moonraker to receive 
updates:

    [update_manager led_effect]
    type: git_repo
    path: ~/klipper-led_effect
    origin: https://github.com/julianschill/klipper-led_effect.git
    is_system_service: False

## Uninstall

Remove all led_effect definitions in your Klipper configuration and the updater
section in the Moonraker configuration. Then run the script to remove the link:

    cd ~
    cd klipper-led_effect
    ./install-led_effect.sh -u

If your directory structure differs from the usual setup you can configure the
installation script with parameters:

    ./install-led_effect.sh -u [-k <klipper path>] [-s <klipper service name>] [-c <configuration path>]

If that fails, you can delete the link in Klipper manually:

    rm ~/klipper/klippy/extras/led_effect.py

Delete the repository (optional):

    cd ~
    rm -rf klipper-led_effect

## Configuration

Documentation can be found [here](docs/LED_Effect.md).

## Instances (Template-Based Multi-LED Support)

The `instances` parameter allows a single effect definition to automatically generate multiple independently controllable effects, each targeting a different LED chain. This eliminates config duplication when multiple LED strips share the same visual behavior.

### Basic Example

Instead of duplicating an effect for each LED strip:

```ini
# Before: duplicated config for each strip
[led_effect left_breathing]
autostart: false
frame_rate: 10
leds:
    neopixel:left_strip
layers:
    breathing 3 1 top (1,0,0)

[led_effect right_breathing]
autostart: false
frame_rate: 10
leds:
    neopixel:right_strip
layers:
    breathing 3 1 top (1,0,0)
```

Use `instances` to define it once:

```ini
# After: single template generates both effects
[led_effect breathing]
instances:
    left = neopixel:left_strip
    right = neopixel:right_strip
autostart: false
frame_rate: 10
layers:
    breathing 3 1 top (1,0,0)
```

This generates two effects: `left_breathing` and `right_breathing`, each independently controllable:

```
SET_LED_EFFECT EFFECT=left_breathing
SET_LED_EFFECT EFFECT=right_breathing STOP=1
```

### Multi-Toolhead Printer Example

For printers with multiple toolheads sharing the same LED layout (e.g. Voron with toolchanger), `instances` dramatically reduces config size. A 4-toolhead setup with 10+ effects goes from ~400 lines to ~80:

```ini
# Logo LEDs (1-8) — breathing red when busy
[led_effect logo_busy]
instances:
    t0 = neopixel:T0 (1-8)
    t1 = neopixel:T1 (1-8)
    t2 = neopixel:T2 (1-8)
    t3 = neopixel:T3 (1-8)
autostart: false
frame_rate: 10
layers:
    breathing 3 1 top (1,0,0)

# Nozzle LEDs (9-10 RGBW) — solid white when printing
[led_effect nozzle_printing]
instances:
    t0 = neopixel:T0 (9-10)
    t1 = neopixel:T1 (9-10)
    t2 = neopixel:T2 (9-10)
    t3 = neopixel:T3 (9-10)
autostart: false
frame_rate: 10
layers:
    static 1 0 top (0,0,0,1.0)

# Rainbow effect on full strip
[led_effect rainbow]
instances:
    t0 = neopixel:T0
    t1 = neopixel:T1
    t2 = neopixel:T2
    t3 = neopixel:T3
autostart: false
frame_rate: 10
layers:
    gradient 0.3 1 add (1.0,0.0,0.0),(0.0,1.0,0.0),(0.0,0.0,1.0)
```

Generated effects follow the naming pattern `{prefix}_{template_name}`:

```
SET_LED_EFFECT EFFECT=t0_logo_busy        # Red breathing on T0 logo
SET_LED_EFFECT EFFECT=t2_nozzle_printing  # White nozzle on T2
SET_LED_EFFECT EFFECT=t3_rainbow          # Rainbow on T3 full strip
STOP_LED_EFFECTS                          # Stops all effects globally
```

### Automation Macro Example

Pair with gcode macros for automatic per-tool LED control:

```ini
[gcode_macro _START_TOOL_EFFECT]
gcode:
    {% set tool = params.TOOL|default("t0") %}
    {% set effect = params.EFFECT|default("logo_standby") %}
    SET_LED_EFFECT EFFECT={tool}_{effect}

[gcode_macro _STOP_TOOL_EFFECT]
gcode:
    {% set tool = params.TOOL|default("t0") %}
    {% set effect = params.EFFECT|default("logo_standby") %}
    SET_LED_EFFECT EFFECT={tool}_{effect} STOP=1
```

### Parameter Reference

| Parameter | Description |
|-----------|-------------|
| `instances` | Multi-line parameter. Each line: `prefix = led_spec` where prefix is prepended to the effect name and led_spec follows standard `leds:` syntax (e.g. `neopixel:strip (1-8)`) |

**Notes:**
- The `leds:` parameter is not needed when using `instances` — each instance defines its own LED target
- The template section itself does not register a gcode command; only the generated children do
- All other parameters (`autostart`, `frame_rate`, `layers`, `run_on_error`, etc.) are shared across all instances
- Each generated effect is fully independent and can be started/stopped individually
