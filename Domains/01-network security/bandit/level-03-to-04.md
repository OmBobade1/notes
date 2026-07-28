## Level
Bandit3 → Bandit4

## Challenge
The password file lives in a hidden directory/file (dotfile) that doesn't show with a plain `ls`.

## Technique Used
List hidden entries with `ls -a`.

## Commands
```bash
cd inhere
ls -a
cat .hidden
```
