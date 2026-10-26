# Terminal host

!!! info "Since v0.1.0"
    The host pieces and the `terminal` package arrive in **v0.1.0**.

The editing core is pure: it needs no interpreter and no tty. Everything that touches a
real terminal sits behind the `reline.IO` interface (`GetC`, `UngetC`, `GetScreenSize`,
`Write`, `MoveCursorColumn`, `EraseAfterCursor`, `ClearScreen`). A `NullIO` (fixed 24×80,
recording) ships for tests and headless use.

## Host pieces in the root package

The host-side parts of MRI that are still pure Go live in
`github.com/go-ruby-reline/reline`:

| API | Port of |
| --- | --- |
| `ReadKeys(config, io)` | `Reline::Core#read_io`: bytes to keys through the keymap, with the `keyseq-timeout` wait on a lone ESC |
| `SetANSIKeyBindings(config)` | `Reline::ANSI#set_default_key_bindings`: cursor keys in CSI and SS3 form, Ctrl/Meta arrows, Delete, Home/End, Page Up/Down, Shift-Tab, M-space, C-x C-x, bracketed paste start |
| `Config.AddDefaultKeyBinding` | the default key-binding table behind the ANSI bindings |
| `key_delete` | the Delete command |
| `MenuLines(list, width)` | `MenuInfo#lines`: the completion menu's columns |
| `LineEditor.TakeMenu()` | hands the pending completion menu to the host |

The root package imports neither `golang.org/x/term` nor `golang.org/x/sys`.

## On a real terminal: `terminal`

```go
import "github.com/go-ruby-reline/reline/terminal"
```

The `terminal` package is the `IO` seam, wired: a `reline.IO` on a real terminal (raw
mode and window size through `golang.org/x/term`; virtual-terminal output enabled on the
Windows console). It puts the terminal in raw mode for one read, decodes the keys, drives
the `LineEditor`, and redraws the edited lines, their prompts and the completion menu
after every key (MRI `Reline.readline` / `Reline.readmultiline` over `Reline::ANSI`).

It supports multiline reads with a termination predicate, per-line prompts, Tab
completion with the menu drawn below the line, history, bracketed paste and Ctrl-L. It
runs on Linux, macOS and Windows 10+. The module requires `golang.org/x/term` and
`golang.org/x/sys`.

```go
t := terminal.New(os.Stdin, os.Stdout) // os.Stdin must be a terminal
for {
    line, err := t.Readline(terminal.Options{
        Prompt:     "> ",
        Terminate:  complete,     // multiline: Enter submits when this accepts the buffer
        Complete:   candidates,   // Tab
        AddHistory: true,
    })
    if err == io.EOF { break }                 // Ctrl-D on an empty line
    if errors.Is(err, terminal.ErrInterrupt) { continue } // Ctrl-C
    run(line)
}
```

`terminal.Options` has the fields `Prompt`, `Terminate`, `PromptProc`, `Complete`,
`AppendCharacter` and `AddHistory`. `Readline` returns `io.EOF` on Ctrl-D or end of
input, and `terminal.ErrInterrupt` on Ctrl-C.

Between two reads the terminal is back in its normal mode, so a program's own output
needs no care. [go-notebook](https://github.com/go-notebook/notebook)'s `notebook irb`
is built on it.
