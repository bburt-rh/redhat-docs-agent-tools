---
name: clip
description: Copy Claude's last response to the system clipboard. Use this skill when the user asks to copy, clip, or add something to the clipboard.
allowed-tools: Bash
---

# Copy to Clipboard

Copy Claude's last response to the system clipboard.

## Prerequisites

- One of the following clipboard utilities must be available:
  - `xclip` (X11)
  - `xsel` (X11)
  - `wl-copy` (Wayland)
  - `pbcopy` (macOS)

## Instructions

1. Identify Claude's last response text (the content immediately preceding the user's clipboard request).
2. Copy that text to the system clipboard using the appropriate clipboard command for the platform.

### Detect the clipboard command

Try commands in this order:

1. If on macOS: `pbcopy`
2. If `wl-copy` is available: `wl-copy`
3. If `xclip` is available: `xclip -selection clipboard`
4. If `xsel` is available: `xsel --clipboard --input`

### Copy to clipboard

Pipe the last response text into the detected clipboard command:

```bash
echo -n '<last_response_text>' | <clipboard_command>
```

Use `printf '%s'` instead of `echo -n` if the text contains escape sequences or special characters. Always single-quote or properly escape the content to avoid shell interpretation.

### Confirm

After copying, confirm to the user that the content has been copied to the clipboard.
