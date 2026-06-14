---
tags: [terminal, shell, macos]
---
# Terminal

## Common Commands

```bash
ls              # List files (no hidden)
ls -a           # List files including hidden
touch <file>    # Create a file
rm <file>       # Remove a file
rm -r <dir>     # Remove directory recursively
cd ~/           # Go to home directory
cp              # Copy files or directories
clear           # Clear terminal
```

## Environment Variables

**Check:**
```bash
printenv                      # See all env vars
printenv MY_VAR               # See a specific one
echo $MY_VAR                  # Alternative
printenv | grep -i digital    # Search by partial name (-i = case-insensitive)
```

**Set temporarily** (current session only — gone when terminal closes):
```bash
export MY_VAR="value"
```

**Set permanently** — add to your shell config:
```bash
# zsh (default on modern Macs)
echo 'export MY_VAR="value"' >> ~/.zshrc
source ~/.zshrc

# bash
echo 'export MY_VAR="value"' >> ~/.bash_profile
source ~/.bash_profile
```

Not sure which shell you're using?
```bash
echo $SHELL
```
