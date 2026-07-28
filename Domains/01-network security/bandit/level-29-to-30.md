## Level
Bandit29 → Bandit30

## Challenge
The password isn't on the default branch; it's on another branch of the repository.

## Technique Used
List all branches and check each one's content.

## Commands
```bash
git clone ssh://bandit29-git@localhost:2220/home/bandit29-git/repo /tmp/repo29
cd /tmp/repo29 && git branch -a
git checkout <other-branch> && cat README.md
```
