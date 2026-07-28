## Level
Bandit31 → Bandit32

## Challenge
The repo's `.gitignore` blocks a specific file; you must force-add and push that exact file to receive the password.

## Technique Used
Create the required file with the required content, force-add it past `.gitignore`, commit, and push.

## Commands
```bash
git clone ssh://bandit31-git@localhost:2220/home/bandit31-git/repo /tmp/repo31
cd /tmp/repo31 && echo 'May I come in?' > key.txt
git add -f key.txt
git commit -m 'add key.txt'
git push origin master
```
