## Level
Bandit9 → Bandit10

## Challenge
The password is embedded in a file full of binary/garbage data, preceded by several `=` characters.

## Technique Used
Use `strings` to pull printable text out of a binary file, optionally piping to `grep`.

## Commands
```bash
strings data.txt | grep '='
```
