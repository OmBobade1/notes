## Level
Bandit22 → Bandit23

## Challenge
The cron script writes its output to a filename derived from `md5sum` of the current date and username, rather than a fixed path.

## Technique Used
Read the script to see how it builds the filename, reproduce that computation, then read the resulting file.

## Commands
```bash
cat /usr/bin/cronjob_bandit23.sh
echo "$(whoami)$(date +%s -d "$(date +%Y-%m-%d)")" | md5sum
cat /tmp/<computed-md5>.txt
```
