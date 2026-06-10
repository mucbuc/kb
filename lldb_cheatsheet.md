# LLDB Cheatsheet

## Starting LLDB

```bash
lldb ./your_executable      # Launch an executable
lldb -p <pid>               # Attach to a running process
```

---

## Breakpoints

| Command | Shorthand | Description |
|---|---|---|
| `breakpoint set --name main` | `b main` | Break at function by name |
| `breakpoint set --file foo.c --line 42` | `b foo.c:42` | Break at file/line |
| `breakpoint list` | `br l` | List all breakpoints |
| `breakpoint delete 1` | `br del 1` | Delete breakpoint #1 |

---

## Execution Control

| Command | Shorthand | Description |
|---|---|---|
| `run` | `r` | Start the process |
| `continue` | `c` | Continue after a pause |
| `next` | `n` | Step over (don't enter calls) |
| `step` | `s` | Step into (enter function calls) |
| `finish` | | Step out of current function |
| `kill` | | Stop the process |

---

## Stack Trace

| Command | Shorthand | Description |
|---|---|---|
| `thread backtrace` | `bt` | Show stack trace for current thread |
| `bt all` | | Show stack trace for all threads |
| `frame select 2` | `f 2` | Switch to frame #2 |
| `frame info` | | Show current frame details |

---

## Inspecting Variables

| Command | Shorthand | Description |
|---|---|---|
| `frame variable` | `fr v` | Show all local variables |
| `frame variable myVar` | `fr v myVar` | Show a specific variable |
| `expression myVar` | `p myVar` | Evaluate and print an expression |
| `po myObject` | | Print object (calls description/toString) |

---

## Misc

| Command | Shorthand | Description |
|---|---|---|
| `help <command>` | | Show docs for a command |
| `quit` | `q` | Exit LLDB |

---

> **Tip:** LLDB accepts unique prefix abbreviations for most commands (e.g. `br s -n main`).
> When in doubt, use `help <command>` for full inline documentation.
