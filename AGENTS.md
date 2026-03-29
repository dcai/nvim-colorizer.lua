# AGENTS.md

## Project Overview

nvim-colorizer.lua is a high-performance color highlighter for Neovim written in LuaJIT. It parses color codes (hex, named colors, CSS functions) in buffers and applies real-time syntax highlights. Zero external dependencies.

Requires Neovim >= 0.4.0 and `set termguicolors`.

## Testing

There is no automated test suite. Manual verification uses `test/expectation.txt`:

1. Open any file in Neovim
2. Run `:ColorizerAttachToBuffer` or `lua require'colorizer'.attach_to_buffer(0, {css=true})`
3. Paste SUCCESS cases from `test/expectation.txt` -- they should highlight
4. Paste FAIL cases -- they should NOT highlight

The trie can be tested standalone:

```bash
nvim --headless -c "luafile test/print-trie.lua" -c "q"
```

There is no linter, formatter, or CI configuration.

## Architecture

### Source Files

| File                     | Role                                                                                                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin/colorizer.vim`   | Entry point. Defines `:ColorizerAttachToBuffer`, `:ColorizerDetachFromBuffer`, `:ColorizerReloadAllBuffers`, `:ColorizerToggle` commands. Guards with `g:loaded_colorizer`. |
| `lua/colorizer.lua`      | Main module. Color parsing, buffer highlighting, attachment lifecycle, user API.                                                                                            |
| `lua/colorizer/nvim.lua` | Metatable-based convenience wrapper around `vim.api`. Provides `nvim.ex`, `nvim.fn`, `nvim.bo`, `nvim.o`, etc. as shorthand.                                                |
| `lua/colorizer/trie.lua` | C FFI trie implementation via `ffi.cdef`. Used for fast color name prefix matching. Manages its own memory with `malloc`/`free` and has a `__gc` finalizer.                 |

### How Highlighting Works

1. **`setup(filetypes, default_options)`** -- registers `FileType` autocmds. Merges per-filetype options with defaults.
2. **`attach_to_buffer(buf, options)`** -- calls `rehighlight_buffer` for the full buffer, then registers `nvim_buf_attach` with `on_lines`/`on_detach` callbacks for incremental updates.
3. **`highlight_buffer(buf, ns, lines, line_start, options)`** -- iterates each character position, calling a compiled matcher function. On match, creates a Neovim highlight group and applies it via `nvim_buf_add_highlight`.

### Matcher System

Matchers are parser functions with signature `(line, i) -> length | nil, rgb_hex | nil`. They are combined via `compile_matcher` into a single closure. Matchers are cached by a bitmask key (`MATCHER_CACHE`) derived from which options are enabled.

Parser functions:

- `color_name_parser` -- uses `COLOR_TRIE` for longest-prefix matching of named colors (e.g. "Blue")
- `rgb_hex_parser` -- handles `#RGB`, `#RRGGBB`, `#RRGGBBAA`
- `css_function_parser` / `rgb_function_parser` / `hsl_function_parser` -- dispatch via small tries for `rgb()`, `rgba()`, `hsl()`, `hsla()`

### Byte Classification

`BYTE_CATEGORY` is a 256-entry FFI array where each byte encodes: bit flags for digit/alpha/hex, and the hex value in the upper nibble. Used by `byte_is_hex`, `byte_is_alphanumeric`, and `parse_hex` to avoid repeated `string.byte` comparisons.

### Highlight Caching

Highlight groups are created lazily and cached in `HIGHLIGHT_CACHE` keyed by `{mode}_{rgb_hex}`. Group names follow the pattern `colorizer_{mb|mf}_{rrggbb}`. Background mode auto-selects black/white foreground based on perceived luminance.

## Key Conventions

- LuaJIT FFI is used for performance-critical paths (trie, byte classification). Do not replace with pure Lua without profiling.
- Semicolons are used as statement separators in table constructors and return tables throughout.
- The `nvim` module (`colorizer/nvim.lua`) wraps all `vim.api` calls -- use it rather than calling `vim.api` directly in new code within this plugin.
- Buffer attachment state is tracked in `BUFFER_OPTIONS[buf]`. Setting it to `nil` signals the `on_lines` handler to detach.
