## Level
Bandit21 → Bandit22

## Challenge
A cron job runs periodically as another user and writes output somewhere readable.

## Technique Used
Inspect the cron configuration and the script it runs to find where the password ends up.

## Commands
```bash
cat /etc/cron.d/cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
cat /tmp/<output-file-the-script-writes>
```
