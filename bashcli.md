[Home](index.md)

# Useful Bash CLI Shortcuts
---
- `Ctrl + a` : Go to start of the command line
- `Ctrl + e` : Go to the end of the command line
- `Ctrl + k` : Delete from the cursor to the end of the command line
- `Ctrl + u` : Delete from the cursor to the start of the command line
- `Ctrl + w` : Delete the word just before the cursor, `Ctrl +y` will undo this.
- `Esc - t`  : will swap the two previous words, for example `man whatis` will become `whatis man`
- `Esc - b`  : Move back one word
- `Esc - f`  : Move forward one word
- `!!`: Rerun the last command
- `!$` passes the argument of the last command in history. For example, if my last command was `ls /usr/share/doc`, I can then do a `cd !$` which will result in the execution of `cd /usr/share/doc`. Another shotcut to do this is `Esc - .`
- `^first_string^second_string`: If I executed `echo hello world` and later want to replace `h` with `H`, I can do a `^he^He` which will replace the first character with the second and execute the command.