## Level
Bandit4 → Bandit5

## Challenge
A directory has ~10 files, only one of which contains human-readable ASCII text.

## Technique Used
Use `file` to inspect the type of every file, or `file *` to check them all at once.

## Commands
```bash
cd inhere
file ./*
cat ./-file07   # whichever file reports 'ASCII text'
```
