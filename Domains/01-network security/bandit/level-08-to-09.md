## Level
Bandit8 → Bandit9

## Challenge
A large file has many duplicate lines; the password is the one line that appears exactly once.

## Technique Used
Sort the file so duplicates are adjacent, then use `uniq -u` to print lines with no duplicates.

## Commands
```bash
sort data.txt | uniq -u
```
