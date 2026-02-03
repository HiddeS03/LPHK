# LPHK - LaunchPad HotKey Copilot Instructions

## Project Overview

LPHK is a macro scripting system for Novation Launchpad controllers, enabling them to function as scriptable, general-purpose macro keyboards. It uses a custom scripting language called **LPHKscript** (similar to DuckyScript) and provides a GUI for managing scripts, colors, and layouts.

**Current Version:** 0.3.1  
**Primary Language:** Python 3  
**Target Platforms:** Windows (primary), Linux (weak support), macOS (untested)

## Core Architecture

### Application Structure

```
LPHK.py          - Main entry point, initialization, platform detection
window.py        - Tkinter GUI and main event loop
lp_events.py     - Launchpad event handling and button press detection
scripts.py       - Script execution engine and scheduling system
lp_colors.py     - Color management for Launchpad buttons
files.py         - Layout/script save/load functionality
kb.py            - Keyboard input abstraction layer
ms.py            - Mouse control using pynput
sound.py         - Sound playback using pygame.mixer
logger.py        - Dual stdout/stderr logging to file
parse.py         - UNUSED variable parsing module
```

### Platform-Specific APIs

- `system_apis/keyboard_win.py` - Windows keyboard control using ctypes/WinAPI
- `system_apis/keyboard_unix.py` - Unix/Linux keyboard control using pynput

### Key Modules

- **launchpad.py** - External dependency from FMMT666's library for Launchpad communication
- **pygame** - Used for MIDI and sound
- **tkinter** - Cross-platform GUI
- **pynput** - Cross-platform keyboard/mouse control
- **pyautogui** - Keyboard/mouse automation fallback

## Critical Patterns & Conventions

### Threading Model

- **Synchronous Scripts**: Single-threaded queue system. Only one runs at a time, others are scheduled.
- **Asynchronous Scripts**: Scripts with `@ASYNC` header run in background threads.
- Each button (x, y) has its own thread stored in `scripts.threads[x][y]`
- Threads have a `kill` event flag for graceful termination
- Use `safe_sleep()` for delays that check kill flag periodically (0.025s intervals)

### Coordinate System

- 9x9 grid (0-8, 0-8) representing physical Launchpad buttons
- **Never use 1-indexed coordinates** - always 0-indexed
- Special position (8, 0) is typically reserved for status indicators

### Color Handling

- Colors are stored as RGB lists: `[r, g, b]` (0-255 each)
- Some Launchpad models only support RG (red-green), not full RGB
- Legacy code converter: `code_to_RGB()` in `lp_colors.py`
- Always use `lp_colors.setXY()` and `lp_colors.updateXY()` to change button colors

### Global State Management

- **Critical globals** in scripts.py:
  - `threads` - 9x9 array of thread objects
  - `to_run` - Queue of scheduled scripts
  - `running` - Boolean flag for synchronous script execution
  - `text` - 9x9 array of script text
- **Critical globals** in lp_events.py:
  - `press_funcs` - 9x9 array of callback functions
  - `pressed` - 9x9 array of button states
- **Critical globals** in window.py:
  - `lp_connected` - Connection status
  - `colors_to_set` - 9x9 array of pending color changes

### Path Management

- `PATH` - Resource path (executable directory or script directory)
- `PROG_PATH` - Program file directory
- `USER_PATH` - User data directory (may differ for portable vs installed)
- `IS_PORTABLE` - Boolean flag for portable mode (no USERPATH file)
- `IS_EXE` - Boolean flag for PyInstaller executable
- Always use `os.path.join()` for path construction
- Handle both `/` and `\` path separators (use `os.path.sep`)

## LPHKscript Language

### Script Headers

- `@ASYNC` - Run in background, non-blocking
- `@SIMPLE key` - Shorthand for press-wait-release pattern
- `@LOAD_LAYOUT` - Load a layout file (BETA, undocumented)
- Headers must be on first line (comments/blank lines before allowed)

### Command Categories

#### Utility Commands

`DELAY`, `LABEL`, `GOTO_LABEL`, `REPEAT_LABEL`, `IF_PRESSED_GOTO_LABEL`, `IF_UNPRESSED_GOTO_LABEL`, `WAIT_UNPRESSED`, `RESET_REPEATS`, `SOUND`, `SOUND_STOP`, `WEB`, `WEB_NEW`, `OPEN`, `CODE`

#### Keyboard Commands

`STRING`, `TAP`, `PRESS`, `RELEASE`, `RELEASE_ALL`

#### Mouse Commands

`M_MOVE`, `M_SET`, `M_SCROLL`, `M_LINE`, `M_LINE_MOVE`, `M_LINE_SET`, `M_STORE`, `M_RECALL`, `M_RECALL_LINE`

### Command Parsing Rules

- Commands are case-sensitive and UPPERCASE
- Arguments are space-separated
- Lines starting with `-` are comments
- Empty lines are ignored
- Use `is_ignorable_line()` to check if a line should be skipped

### Valid Command List

Always validate against `VALID_COMMANDS` in scripts.py:

```python
VALID_COMMANDS = ["@ASYNC", "@SIMPLE", "@LOAD_LAYOUT", "STRING", "DELAY",
                  "TAP", "PRESS", "RELEASE", "WEB", "WEB_NEW", "CODE",
                  "SOUND", "SOUND_STOP", "WAIT_UNPRESSED", "M_MOVE",
                  "M_SET", "M_SCROLL", "M_LINE", "M_LINE_MOVE", "M_LINE_SET",
                  "LABEL", "IF_PRESSED_GOTO_LABEL", "IF_UNPRESSED_GOTO_LABEL",
                  "GOTO_LABEL", "REPEAT_LABEL", "IF_PRESSED_REPEAT_LABEL",
                  "IF_UNPRESSED_REPEAT_LABEL", "M_STORE", "M_RECALL",
                  "M_RECALL_LINE", "OPEN", "RELEASE_ALL", "RESET_REPEATS"]
```

## File Formats

### Layout Files (.lpl)

- JSON format with structure:
  ```json
  {
    "version": "FILE_VERSION",
    "buttons": [
      [{"color": [r, g, b], "text": "script"}, ...],
      ...
    ]
  }
  ```
- 9x9 grid of button objects
- Always validate JSON structure
- Legacy format (.LPHKlayout) is auto-converted

### Script Files (.lps)

- Plain text, UTF-8 encoded
- One command per line
- Comments start with `-`
- First non-comment line may be a header

### Sound Files

- Must be in `user_sounds/` directory
- Supported formats: `.wav`, `.flac`, `.ogg`
- pygame.mixer handles playback

## Error Handling

### File Operations

- Always use try-except for JSON operations
- Provide user-friendly error messages via `window.app.popup()`
- Log all errors to console with module prefix: `[module_name] Error message`

### Script Execution

- Scripts should catch their own exceptions and log them
- Use `check_kill(x, y, is_async)` frequently to allow early termination
- Clean up resources (release keys, stop sounds) on script exit

### Launchpad Connection

- Connection can break at any time (USB issues are common)
- Always check `window.lp_connected` before LP operations
- Provide clear instructions for reconnection in error messages

## GUI Conventions

### Window Management

- Main window is `window.root` (Tk instance)
- App instance is `window.app` (Main_Window class)
- Always set window icon using `MAIN_ICON` (.ico for Windows, .gif for others)

### Button Modes

- `edit` - Click to edit script/color
- `move` - Click source, click destination
- `swap` - Click to swap two buttons
- `copy` - Click source, click destination

### Color Indicators

- Red flash (fast) - Script running
- Red pulse (slow) - Script scheduled
- Orange - Function keys primed (hardware limitation)
- `STAT_ACTIVE_COLOR = "#080"` - Connected status
- `STAT_INACTIVE_COLOR = "#444"` - Disconnected status

## Logging Best Practices

### Log Format

```python
print("[module_name] (x, y) Action description")
```

### Log Prefixes

- `[LPHK]` - Main program
- `[scripts]` - Script execution
- `[lp_events]` - Event handling
- `[files]` - File operations
- `[sound]` - Sound playback
- `[window]` - GUI operations

### Log File

- Dual logging to stdout and `LPHK.log` in USER_PATH
- Use `logger.start(LOG_PATH)` to initialize
- Never call `logger.stop()` - let it persist for entire session

## Dependencies & Installation

### Core Dependencies

```
pygame>=2.1.2        # MIDI and sound
tkinter              # GUI (built-in)
pynput>=1.6.8        # Cross-platform input
PyAutoGUI>=0.9.50    # Keyboard/mouse automation
Pillow>=9.3.0        # Image handling
tkcolorpicker>=2.1.3 # Color picker widget
launchpad-py         # From GitHub (FMMT666)
```

### Platform-Specific

- **Windows**: Uses ctypes for WinAPI keyboard control
- **Linux**: Uses pynput and python-xlib
- **Build**: PyInstaller for executables, Inno Setup for Windows installer

## Common Pitfalls

1. **Never modify global state without proper locking** - Threading issues are real
2. **Always validate script text before execution** - Malformed scripts can crash threads
3. **Don't assume Launchpad is connected** - Check `window.lp_connected` first
4. **Path separators** - Use `os.path.join()`, never hardcode `/` or `\`
5. **Color conversions** - Remember some Launchpads only support RG, not RGB
6. **Thread cleanup** - Always set kill flag and wait for thread to exit
7. **Legacy format support** - Always handle both `.lpl` and `.LPHKlayout` files
8. **Coordinate bounds** - Always validate x, y are within [0, 8]

## Testing Considerations

- Test on both Windows and Linux if modifying platform-specific code
- Test with physical Launchpad when possible
- Use dummy connection mode (`connect_dummy()`) for GUI testing
- Test script scheduling with multiple scripts
- Test script killing (tap running button)
- Test USB disconnect/reconnect scenarios

## Code Style

- Use 4 spaces for indentation (not tabs)
- Function names are `snake_case`
- Class names are `PascalCase` (with underscores: `Main_Window`)
- Constants are `UPPER_SNAKE_CASE`
- Module-level variables start lowercase
- Prefer explicit over implicit
- Add comments for complex logic, especially in script execution

## Future Development Notes

- `parse.py` is currently UNUSED - variable system not implemented
- `@LOAD_LAYOUT` header is BETA and undocumented
- PyAutoGUI refactor planned to improve key support
- Auto-update feature planned using VERSION file
- Splash screen waiting on PyInstaller feature

## Security Considerations

- Scripts can execute arbitrary code via `CODE` command
- Scripts can open files/websites
- Scripts have full keyboard/mouse control
- Always validate user input in script editor
- Warn users about running untrusted scripts

## When Modifying:

### Adding New Commands

1. Add to `VALID_COMMANDS` list in scripts.py
2. Implement in `run_script()` main_logic function
3. Add to README.md commands list
4. Add example script in `user_scripts/examples/`
5. Test scheduling behavior (sync vs async)

### Adding New GUI Features

1. All GUI code goes in window.py
2. Use existing image resources in `resources/`
3. Follow Tkinter grid layout pattern
4. Update menu system if adding menu items
5. Test window resizing behavior

### Modifying File Formats

1. Increment `FILE_VERSION` in files.py
2. Maintain backward compatibility
3. Add conversion logic for old formats
4. Update example files
5. Document in README.md

## Key Variables Reference

### scripts.py

- `COLOR_PRIMED = 5` - Red for running scripts
- `COLOR_FUNC_KEYS_PRIMED = 9` - Amber for function keys
- `EXIT_UPDATE_DELAY = 0.1` - Delay before updating button after exit
- `DELAY_EXIT_CHECK = 0.025` - Frequency of kill flag checks

### window.py

- `BUTTON_SIZE = 40` - Canvas button size in pixels
- `HS_SIZE = 200` - Script editor text height
- `V_WIDTH = 50` - Vertical width parameter
- `INDICATOR_BPM = 480` - Flashing speed for indicators
- `DEFAULT_COLOR = [0, 0, 255]` - Blue default for most models
- `MK1_DEFAULT_COLOR = [0, 255, 0]` - Green default for MK1

### lp_events.py

- `RUN_DELAY = 0.005` - Event loop delay (200 FPS)

## Useful Helper Functions

- `kb.sp(name)` - Special key name converter, handles `mouse_` prefix
- `files.load_layout()` - Auto-handles legacy conversion
- `lp_colors.RGB_to_RG()` - Convert RGB to RG for limited models
- `scripts.safe_sleep()` - Sleep with kill flag checking
- `scripts.schedule_script()` - Proper way to start scripts
- `lp_events.bind_func_with_colors()` - Bind function to button with color

## Remember

- This is a **hobby project** with rough edges - some features are incomplete
- USB connection issues are **hardware limitations**, not software bugs
- Linux support is **weak** - most systems cannot run it successfully
- The GUI is intentionally simple - it's a functional tool, not polished software
- Performance matters - the event loop runs at 200 FPS for responsiveness
