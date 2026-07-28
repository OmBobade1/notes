## Level
Bandit30 → Bandit31

## Challenge
The password is stored in an annotated git tag's message, not in any file or branch content.

## Technique Used
List tags and show the tag's metadata.

## Commands
```bash
git clone ssh://bandit30-git@localhost:2220/home/bandit30-git/repo /tmp/repo30
cd /tmp/repo30 && git tag
git show <tag-name>
```
