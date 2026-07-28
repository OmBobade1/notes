## Level
Bandit18 → Bandit19

## Challenge
This user's `.bashrc` immediately logs you out on interactive login, so a normal SSH session never gives you a shell.

## Technique Used
Pass the command to run directly as an argument to `ssh`, which executes it without invoking the interactive logout script.

## Commands
```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
```
