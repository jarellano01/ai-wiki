# Terminal Setup: WezTerm + Zellij + Neovim + Starship + Glow (Claude Code oriented)

Claude Code-oriented terminal setup with Cmd+Click file opening in floating nvim panes.

## What You Get

- **WezTerm** as terminal emulator (Catppuccin Mocha theme, JetBrains Mono font)
- **Zellij** as terminal multiplexer (replaces tmux)
- **Neovim** (kickstart.nvim) as editor
- **Cmd+Click** on any file path in Claude Code output opens it in a floating nvim pane via Zellij
- **Oh My Zsh** + **Starship** prompt
- **fzf** for fuzzy file/history search
- **glow** for terminal markdown rendering with pager

## 1. Install Dependencies

```bash
# macOS (Homebrew)
brew install --cask wezterm
brew install zellij neovim fzf starship glow
brew install --cask font-jetbrains-mono

# Verify
wezterm --version
zellij --version
nvim --version
fzf --version
starship --version
```

## 2. Oh My Zsh + Shell Config

```bash
# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

In `~/.zshrc`, set the theme to empty (Starship handles the prompt) and enable the git plugin:

```bash
ZSH_THEME=""
plugins=(git)
```

Add these at the end of `~/.zshrc`:

```bash
# Claude Code alias
alias cc="claude --dangerously-skip-permissions"

# Starship prompt
eval "$(starship init zsh)"
```

### Starship Config

Create `~/.config/starship.toml`:

```toml
# Minimal starship prompt — keeps it fast and clean
format = """$directory$git_branch$git_status$python$nodejs$character"""

[directory]
truncation_length = 3
truncate_to_repo = true
style = "bold cyan"

[git_branch]
format = "[$branch]($style) "
style = "bold purple"

[git_status]
format = '([$all_status$ahead_behind]($style) )'
style = "bold red"

[python]
format = '[${version}]($style) '
style = "yellow"
detect_files = ["pyproject.toml"]

[nodejs]
format = '[${version}]($style) '
style = "green"
detect_files = ["package.json"]

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"
```

## 3. Neovim Config (kickstart.nvim)

```bash
# Back up existing config if needed
[ -d ~/.config/nvim ] && mv ~/.config/nvim ~/.config/nvim.bak

# Clone kickstart.nvim
git clone https://github.com/nvim-lua/kickstart.nvim ~/.config/nvim

# Launch nvim once to install plugins
nvim --headless +qa 2>/dev/null
nvim  # Open and wait for Lazy to finish installing, then :q
```

Custom plugins go in `~/.config/nvim/lua/custom/plugins/init.lua`.

### Enable gitsigns keymaps

Uncomment this line in `~/.config/nvim/init.lua` (~line 920):

```lua
require 'kickstart.plugins.gitsigns', -- adds gitsigns recommend keymaps
```

This enables inline blame, diff, and hunk navigation.

### Enable permanent inline blame

In `~/.config/nvim/init.lua`, add `current_line_blame = true` to the gitsigns `opts`:

```lua
opts = {
  current_line_blame = true,
  signs = { ... },
},
```

Toggle it off anytime with `Space t b`.

### Show hidden files in Telescope file finder

By default, Telescope's `find_files` picker hides dotfiles (`.github/`, `.gitignore`, `.env.example`, etc.). To show them, add a `pickers` block to the `telescope.setup` call in `~/.config/nvim/init.lua`:

```lua
require('telescope').setup {
  pickers = {
    find_files = {
      hidden = true,
    },
  },
  extensions = {
    ['ui-select'] = { require('telescope.themes').get_dropdown() },
  },
}
```

### Key Neovim Git Shortcuts

| Shortcut | Action |
|----------|--------|
| `Space t b` | Toggle inline blame on every line |
| `Space h b` | Blame current line (detailed) |
| `Space h d` | Diff against index |
| `Space h D` | Diff against last commit |
| `Space h p` | Preview hunk inline |
| `Space h s` | Stage hunk |
| `Space h r` | Reset hunk |
| `] c` / `[ c` | Jump to next/previous git change |

## 4. Zellij Config

Generate the default config, then apply our customizations:

```bash
mkdir -p ~/.config/zellij
zellij setup --dump-config > ~/.config/zellij/config.kdl
```

Edit `~/.config/zellij/config.kdl` and make these changes:

**Add theme at the top** (before `keybinds`):

```kdl
theme "catppuccin-mocha"
```

**Move file picker off Alt+f** so it doesn't conflict with `ToggleFloatingPanes`. Find the `shared_except "locked"` block and change `Alt f` to `Alt Shift f`:

```kdl
shared_except "locked" {
    // ... other bindings ...
    bind "Alt Shift f" {
        LaunchPlugin "filepicker" {
            close_on_selection true
        };
    }
}
```

### Key Zellij Shortcuts

| Shortcut | Action |
|----------|--------|
| `Alt+f` | Toggle floating panes (show/hide) |
| `Alt+n` | New pane |
| `Alt+Shift+f` | File picker |
| `Ctrl+p x` | Close focused pane |
| `Ctrl+p p` | Cycle focus between panes |
| `Ctrl+t n` | New tab |
| `Ctrl+t 1-9` | Go to tab N |
| `Ctrl+g` | Lock mode (pass keys through to app) |
| `Ctrl+q` | Quit Zellij |

## 5. WezTerm Config

Create `~/.wezterm.lua`:

```lua
local wezterm = require 'wezterm'
local act = wezterm.action
local config = wezterm.config_builder()

-- Appearance
config.color_scheme = 'Catppuccin Mocha'
config.font = wezterm.font('JetBrains Mono', { weight = 'Medium' })
config.font_size = 14.0
config.window_decorations = 'TITLE | RESIZE'
config.window_padding = { left = 4, right = 4, top = 4, bottom = 4 }
config.hide_tab_bar_if_only_one_tab = true
config.native_macos_fullscreen_mode = true

-- Disable WezTerm's own multiplexing (Zellij handles this)
config.default_prog = { '/bin/zsh', '-l' }

-- Let CMD+click bypass Zellij's mouse capture so WezTerm handles hyperlinks
config.bypass_mouse_reporting_modifiers = 'CMD'

-- Keep default hyperlink rules for URLs
config.hyperlink_rules = wezterm.default_hyperlink_rules()

-- File extensions to open in nvim (not binary/media files)
local editable_extensions = {
  ts=true, tsx=true, js=true, jsx=true, py=true, lua=true, rs=true, go=true,
  rb=true, java=true, c=true, cpp=true, h=true, hpp=true, cs=true,
  json=true, yaml=true, yml=true, toml=true, kdl=true, ini=true, conf=true,
  md=true, txt=true, css=true, scss=true, html=true, xml=true, svg=true,
  sh=true, bash=true, zsh=true, fish=true, sql=true, graphql=true, gql=true,
  vim=true, lock=true, env=true, gitignore=true, dockerignore=true,
  dockerfile=true, makefile=true,
}

local function is_editable(path)
  local ext = path:match('%.(%w+)$')
  if ext then
    return editable_extensions[ext:lower()] or false
  end
  local base = path:match('[^/]+$')
  if base then
    return editable_extensions[base:lower()] or false
  end
  return false
end

-- Intercept file:// URIs -> open editable files in nvim via zellij floating pane
--
-- Claude Code emits OSC 8 explicit hyperlinks with file:// URIs for file paths.
-- This handler intercepts those and opens them in nvim inside a Zellij floating
-- pane instead of the default OS app.
--
-- IMPORTANT: The Zellij session name is hardcoded below. Change it to match
-- your session name, or use a dynamic lookup.
wezterm.on('open-uri', function(window, pane, uri)
  wezterm.log_info('open-uri: ' .. uri)

  if uri:find('^file://') then
    local path = uri:gsub('^file://', '')
    local base_path, line = path:match('^(.+):(%d+)$')
    if not base_path then
      base_path = path
    end

    if is_editable(base_path) then
      -- *** CHANGE THIS to your Zellij session name ***
      local session = 'hl'
      local info = pane:get_user_vars()
      if info and info.ZELLIJ_SESSION_NAME then
        session = info.ZELLIJ_SESSION_NAME
      end

      local shell_cmd = 'zellij --session ' .. session .. ' action new-pane --floating -- nvim'
      if line and line ~= '' then
        shell_cmd = shell_cmd .. ' +' .. line
      end
      shell_cmd = shell_cmd .. ' ' .. base_path

      wezterm.log_info('Opening in nvim: ' .. shell_cmd)
      wezterm.run_child_process({ '/bin/zsh', '-lc', shell_cmd })
      return false
    end
  end
end)

-- Pass Shift+Enter through to applications (needed for Claude Code in Zellij)
config.keys = {
  { key = 'Enter', mods = 'SHIFT', action = act.SendString('\x1b[13;2u') },
}

-- CMD+click opens hyperlinks
config.mouse_bindings = {
  {
    event = { Up = { streak = 1, button = 'Left' } },
    mods = 'CMD',
    action = act.OpenLinkAtMouseCursor,
  },
}

return config
```

## 6. fzf (Fuzzy Finder)

fzf adds fuzzy search for files, command history, and directories directly in the shell.

Add to `~/.zshrc`:

```bash
# fzf — fuzzy finder (Ctrl+R for history, Ctrl+F for files)
source <(fzf --zsh)
bindkey -r '^T'
bindkey '^F' fzf-file-widget
```

Reload with `source ~/.zshrc`.

### Key fzf Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+F` | Fuzzy file search (rebound from default `Ctrl+T`) |
| `Ctrl+R` | Fuzzy command history search |
| `Alt+C` | Fuzzy cd into a directory |

> **Note:** Inside Zellij, `Ctrl+R` and `Alt+C` work out of the box. `Ctrl+F` requires that Zellij doesn't have a conflicting binding on that key.

## 7. Glow (Markdown Renderer)

[Glow](https://github.com/charmbracelet/glow) renders markdown beautifully in the terminal — useful for reading docs, READMEs, and Claude Code output files without leaving the CLI.

```bash
brew install glow
```

### Glow Config

The config file lives at `~/Library/Preferences/glow/glow.yml` (macOS) or `~/.config/glow/glow.yml` (Linux). Generate it with `glow config` or create it manually:

```yaml
# style name or JSON path (default "auto")
style: "dark"
# mouse wheel support (TUI-mode only)
mouse: true
# use pager to display markdown
pager: true
# at which column should we word wrap?
width: 80
# show all files, including hidden and ignored.
all: false
# show line numbers (TUI-mode only)
showLineNumbers: false
# preserve newlines in the output
preserveNewLines: false
```

With `pager: true`, running `glow file.md` pipes through a pager automatically — no need to pass `-p` every time. The `dark` style matches Catppuccin Mocha and other dark terminal themes; use `"light"` for light themes or `"auto"` to detect automatically.

## 8. Usage

1. Open WezTerm
2. Start a Zellij session: `zellij attach hl --create`
3. Run Claude Code in a pane
4. **Cmd+Click** any file path in Claude Code output — it opens in a floating nvim pane
5. Edit the file, `:wq` to save and close
6. If you click outside the floating pane, press `Alt+f` to bring it back

## How Cmd+Click Works

Claude Code emits [OSC 8 hyperlinks](https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda) with `file://` URIs for file paths in its output. WezTerm's `open-uri` event handler intercepts these before macOS opens them in the default app. For editable file types, it runs `zellij action new-pane --floating -- nvim <path>` to open the file in a floating Zellij pane.

Key pieces that make this work:
- `bypass_mouse_reporting_modifiers = 'CMD'` — prevents Zellij from swallowing the Cmd+Click
- `zellij --session <name>` — required because `wezterm.run_child_process` runs outside the Zellij session and needs to know which session to target
- `--floating` flag — opens nvim in a floating pane overlay instead of splitting the layout

## Debugging

Open the WezTerm debug overlay with `Ctrl+Shift+L` to see `open-uri` log messages. If clicking opens files in the wrong app, the handler isn't matching — check:
- Is the file extension in the `editable_extensions` table?
- Is the Zellij session name correct?
- Is the URI a `file://` URI? (check the debug log)

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Cmd+Click opens file in Xcode/Cursor | The `open-uri` handler isn't matching. Check debug overlay (`Ctrl+Shift+L`) for the URI format and verify file extension is in the editable list. |
| Cmd+Click does nothing at all | Zellij is capturing the mouse. Verify `bypass_mouse_reporting_modifiers = 'CMD'` is set. |
| Floating pane appears but nvim errors | Check that `nvim` is in your PATH from a login shell (`/bin/zsh -l`). |
| Wrong Zellij session | Update the `session` variable in `.wezterm.lua` to match your `zellij attach <name>` session name. |
| Zellij status bar has wrong colors | Verify `theme "catppuccin-mocha"` is at the top of `~/.config/zellij/config.kdl`. |
| Shift+Enter doesn't work in Claude Code | WezTerm + Zellij can swallow Shift+Enter. The `config.keys` entry above sends the correct CSI u escape sequence (`\x1b[13;2u`) so it passes through Zellij to Claude Code. |
