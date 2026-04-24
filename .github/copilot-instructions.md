# TinyFugue Copilot Instructions

## Project Overview

**TinyFugue** (tf) is a programmable MUD (Multi-User Dungeon) client written in C. It's a mature, cross-platform project supporting UNIX-like systems (including macOS), OS/2, and Win32. The codebase dates from 1993 with significant development through 2007.

**Key Goals:**
- Provide a rich, extensible MUD client with a scripting language
- Support multiple platforms with minimal code duplication
- Maintain backward compatibility across versions

## Build System

### Build Environment

The project uses **GNU autoconf** for UNIX-like systems. Key tools:
- `./configure` - Auto-configuration script that detects system capabilities
- `make` - Compile and install the project
- The build system determines platform-specific features (SSL, IPv6, termcap, etc.)

### Build & Install Commands

**For macOS/UNIX:**
```bash
./configure [options]
make install
```

**Useful configure options:**
- `--enable-version` - Embed version number in executable and library names
- `--enable-symlink[=NAME]` - Create symlink to the executable
- `--enable-core` - Enable debugging: core files, debug symbols (-g), disable optimization
- `--disable-ssl` - Disable SSL support
- `--enable-inet6` - Enable IPv6 (may be broken)
- `--disable-termcap` - Use hardcoded vt100 codes instead of termcap
- `--enable-termcap=LIB` - Specify termcap library (e.g., ncurses)
- `--disable-history` - Disable /recall and history features
- `--disable-process` - Disable /quote and /repeat
- `--disable-float` - Disable floating-point arithmetic
- `--with-inclibpfx=DIRS` - Specify include/lib search paths

**Useful make targets:**
- `make all` - Compile only, don't install
- `make install` - Compile and install
- `make clean` - Remove object files and build artifacts
- `make uninstall` - Remove installed files

### Build Configuration Hierarchy

The project uses a hierarchical configuration approach:
1. Platform-specific: `unix/Makefile`, `os2/`, `win32/` directories
2. Root `Makefile` directs to the appropriate platform
3. The `configure` script generates platform-specific Makefiles

### Environment Variables for Build

```bash
CC           # C compiler (default: gcc or cc)
CFLAGS       # Compiler flags (default: -g -O2 for gcc, -g for others)
CPPFLAGS     # Preprocessor flags (e.g., -I for include paths)
LIBS         # Linker flags (-l and -L options)
```

Example:
```bash
env CC=clang CFLAGS="-g -O2" ./configure
```

## Source Code Organization

### Core Architecture

**Main execution flow:** `src/main.c` → initialization → `src/socket.c` → main event loop

**Key subsystems by module:**

| Module | Purpose |
|--------|---------|
| `main.c` | Initialization, configuration loading |
| `socket.c` | Network I/O and main event loop (select-based multiplexing) |
| `command.c` | Built-in command dispatch and execution |
| `macro.c` | User-defined macros/aliases |
| `expand.c` | Macro/variable expansion and substitution |
| `expr.c` | Expression evaluation (scripting language) |
| `keyboard.c` | Keyboard input handling |
| `output.c` | Output rendering and display |
| `history.c` | Command/recall history |
| `dstring.c` | Dynamic strings with attributes |
| `attr.c` | Text attributes (colors, bold, etc.) |
| `world.c` | World/connection management |
| `pattern.c` | Pattern matching and triggers |
| `search.c` | Pattern search engine |
| `variable.c` | Variable management |
| `tty.c` | Terminal control and termcap/hardcoded vt100 codes |

### Dynamic Strings (dstring)

TinyFugue uses a custom dynamic string type (`String`) throughout:
```c
typedef struct String {
    char *data;          // actual text
    int len;             // length (excluding NUL)
    int size;            // allocated size
    short links;         // reference count
    attr_t attrs;        // line-wide attributes (colors, etc.)
    cattr_t *charattrs;  // per-character attributes
    struct timeval time; // timestamp
};
```

**Key patterns:**
- Strings are reference-counted (`links` field)
- Functions expect `String*` or `conString*` (const String)
- Many functions return new strings that must be freed via `free_string()`

### List Files (*.h files with special purposes)

Several header files are generated/managed lists:
- `cmdlist.h` - Built-in command definitions (sorted by name)
- `funclist.h` - Built-in function definitions
- `hooklist.h` - Hook/trigger points
- `keylist.h` - Default key bindings
- `enumlist.h` - Enumeration constants
- `opcodes.h` - VM opcodes for compiled expressions

**Convention:** When adding new commands, functions, or hooks, add them to the corresponding list file and ensure they're sorted alphabetically (especially important for `cmdlist.h` due to binary search).

### Platform-Specific Code

Platform differences are isolated in directories:
- `src/` - Core platform-neutral code
- `unix/` - UNIX/macOS specific (`tfconfig.h` from configure, `unix.mak`)
- `os2/`, `win32/` - OS/2 and Windows specific

**Key platform abstractions:**
- `port.h` - Platform detection and abstractions
- `tty.c` - Terminal control (termcap vs. hardcoded vt100)
- `signals.c` - Signal handling (UNIX-specific)
- `process.c` - Process spawning (/quote and /repeat commands)

### Library Files (tf-lib)

The `tf-lib/` directory contains the standard library of TinyFugue scripts:
- `stdlib.tf` - Core standard library (always loaded)
- Utility scripts like `complete.tf`, `color.tf`, `alias.tf`, etc.
- These are installed alongside the binary and loaded at runtime

## Key Conventions

### Code Style

- **Comments:** Concise, brief comments explaining "why" not "what"
- **Naming:** lowercase with underscores for functions/variables; UPPERCASE for macros
- **Macro pattern:** Many features are compile-time configurable via preprocessor flags (NO_HISTORY, NO_PROCESS, NO_FLOAT)
- **Assertions:** Uses `assert()` and custom error checking

### Memory Management

- **Strings:** Use `malloc_string()`, `free_string()`, `alloc_string()` from `dstring.c`
- **Arrays/structures:** Manual malloc/free with error checking
- **Reference counting:** The `String` type uses link counting; increment `s->links` before storing, call `free_string()` when done
- **No memory pools:** TinyFugue allocates directly from the heap

### Variable and Command Naming

- **Built-in commands:** Prefixed with `/` (e.g., `/load`, `/addworld`)
- **Variables:** Prefixed with `%` (e.g., `%var_name`)
- **Macros:** User-defined functions defined with `/def` command
- **Binary search assumption:** Commands in `cmdlist.h` MUST be sorted alphabetically (code assumes this)

### Error Handling

- **Pattern:** Functions return `NULL`, `0`, or `-1` for errors (check documentation)
- **String errors:** Often returned as error text in a `String*` that must be freed
- **Signals:** UNIX signal handlers (SIGWINCH, SIGTERM, etc.) set volatile flags that are checked in main loop
- **No exceptions:** C project; errors propagate via return values

### Testing & Debugging

- **No automated tests:** Manual testing via TinyFugue's built-in `/help` and interactive testing
- **Debugging flags:** Compile with `--enable-core` to get debug symbols
- **Core files:** Platform-dependent; may be disabled/restricted on some systems
- **Logging:** Use output functions like `oputs()`, `oprintf()` for visible output

### Input Expansion Sequence

When user input is processed:
1. **Macro expansion** (`src/expand.c`) - Replace `%` variables and `{}` braces
2. **Substitution** (`src/expand.c`) - Process escape sequences
3. **Command dispatch** (`src/command.c`) - Look up command or treat as text
4. **Expression evaluation** (`src/expr.c`) - If needed by command (e.g., `/if` conditions)

This sequence is important for understanding variable/macro scoping.

### Pattern/Trigger System

Patterns can be:
- **Simple strings** - Exact match
- **PCRE regular expressions** - Full regex support (embedded PCRE library in `src/pcre-2.08/`)
- **Attributes** - Matched against text color/formatting

Triggers are defined via `/def` and executed when patterns match received text.

## File Locations

After installation, the default layout is:
- **macOS/UNIX:** `${PREFIX}/bin/tf` and `${PREFIX}/share/tf-lib/`
  - Default prefix: `/usr/local` (if writable) or `$HOME`
- **Library files:** Environment variable `TFLIBDIR` or `-L` flag overrides defaults
- **Help files:** Generated from documentation, stored in library directory

## Important Notes

- **Restricted mode:** Features can be disabled via `/restrict SHELL`, `/restrict FILE`, `/restrict WORLD` in `%{TFLIBDIR}/local.tf`
- **Compression:** Library and help files can be gzip-compressed (configure `COMPRESS_SUFFIX` and `COMPRESS_READ`)
- **Proxy support:** Supports SOCKS proxy and generic proxy via `%proxy_host` variable
- **Terminal compatibility:** Falls back to hardcoded vt100 codes if termcap unavailable
- **No built-in tests:** The project relies on manual testing and integration testing via the interactive client
- **Version embedding:** Use `--enable-version` to embed version in executable name; useful for keeping multiple versions installed

## Contributing Changes

When modifying the codebase:

1. **For new commands:** Add entry to `src/cmdlist.h` (keep alphabetically sorted), implement handler in appropriate `.c` file
2. **For new built-in functions:** Add to `src/funclist.h` and implement in `expr.c` or related module
3. **For new variables:** Define in `src/variable.c` and register appropriately
4. **For platform-specific code:** Place in the `unix/`, `os2/`, or `win32/` directory; use `port.h` for abstractions
5. **For string handling:** Use the `String` type and reference counting conventions
6. **When adding features:** Consider compile-time disable options (via `NO_*` flags) for security/size concerns

## Reference Documentation

- `/help` in TinyFugue - Built-in help system (comprehensive)
- `README` in root - Installation and configuration overview
- `unix/README` - UNIX-specific build/install details
- `COPYING` - GNU GPL v2 license
- Source files - Well-commented with function documentation
