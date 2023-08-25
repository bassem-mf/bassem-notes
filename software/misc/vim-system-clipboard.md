# Access System Clipboard from Vim

Copy & Paste between Vim and other programs.

## Install the Right Vim Package

Make sure that Vim was compiled with clipboard support.

```bash
vim --version | grep clipboard
```

Here is an example output.

```text
+clipboard         +keymap            +printer           +vertsplit
+eval              -mouse_jsbterm     -sun_workshop      +xterm_clipboard
```

If you see `+clipboard` in the output, then your Vim was compiled with clipboard support. If you see `-clipboard` in the output, then your
Vim was Not compiled with clipboard support and cannot access the clipboard.

If you get `-clipboard` in the output then you need to find a package that installs Vim with clipboard support. Look for a package that
installs "GVim" (the GUI version of Vim). On Debian 12 (Bookworm), the package [vim-gtk3](https://packages.debian.org/bookworm/vim-gtk3)
installs GVim and Vim with clipboard support.

## Copy From Vim to Clipboard

1. Make sure you are in "Visual" mode by pressing "ESC" to go to the Normal mode, then pressing "v" to go to the Visual mode.
2. Use the arrow keys to highlight a piece of text.

If your Vim has mouse support, you can drag to select a piece of text instead of steps 1 and 2.

3. Type `"+y`<br />
`"` means that you want to access a register.<br />
`+` is the name of the system clipboard register.<br />
`y` for yank (copy).

4. The text is now in your clipboard and can be pasted in other programs.

## Paste From Clipboard to Vim

1. Make sure you are in "Insert" mode by pressing "ESC" to go to the Normal mode, then pressing "i" to go to the Insert mode.
2. Use the arrow keys to place the cursor where you want to insert the text in the clipboard.
3. Press "CTRL + r" and then "+" to paste from the system clipboard.