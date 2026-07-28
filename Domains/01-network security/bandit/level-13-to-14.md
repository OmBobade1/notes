## Level
Bandit13 → Bandit14

## Challenge
This level provides an SSH private key (`sshkey.private`) instead of a password, used to log in as the next user.

## Technique Used
Restrict the key's permissions, then use `ssh -i` to authenticate with it.

## Commands
```bash
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@localhost -p 2220
```
